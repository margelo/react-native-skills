---
name: react-native-nitro-sqlite
description: Fast embedded SQLite for React Native via react-native-nitro-sqlite (Nitro Modules / JSI, the successor to react-native-quick-sqlite). Covers installation and native config, opening connections, sync vs async queries, the QueryResult shape (results / rows / rowsAffected / insertId / metadata), parameter binding, transactions, batch execution, loading SQL files, attach/detach across databases, the per-database operation queue (and why sync calls throw when the DB is busy), TypeORM integration, error handling with NitroSQLiteError, blobs/ArrayBuffer, FTS5/Geopoly compile flags, system-SQLite-on-iOS, app groups, and migrating from react-native-quick-sqlite.
license: MIT
metadata:
  author: margelo
  scope: react-native-nitro-sqlite
  tags: react-native, sqlite, database, nitro-modules, jsi, async, transactions, batch, typeorm, fts5, blob, quick-sqlite, migration
---

# react-native-nitro-sqlite

A focused reference for AI coding assistants working in a project that uses `react-native-nitro-sqlite`. Answer using the **real APIs** from this library — not invented ones — by routing to the matching `references/*.md` file below.

> This library is the Nitro Modules successor to `react-native-quick-sqlite`. From `9.0.0` on, the package is `react-native-nitro-sqlite`. Current reference version for this skill: **9.6.0**.

## Mental model

`react-native-nitro-sqlite` embeds the SQLite C engine and exposes it to JS over [Nitro Modules](https://nitro.margelo.com/) (JSI) — no bridge, no serialization, direct synchronous access. It is **not** a key-value store; it is real SQL. (For key-value storage use [`react-native-mmkv`](https://github.com/mrousavy/react-native-mmkv) instead.)

The five things that trip people up:

1. **Every operation comes in sync and async form.** `db.execute(...)` runs on the JS thread and returns immediately. `db.executeAsync(...)` runs the SQL on a separate native thread and returns a `Promise`. Prefer **async** for anything non-trivial so you don't block the UI.
2. **A per-database operation queue serializes only `executeBatch`, `executeBatchAsync`, and `transaction`.** `executeBatchAsync` and `transaction` are queued and run **serially**; synchronous `executeBatch` **throws** if the database is busy with a queued/in-progress async batch or transaction: `"Database <name> is busy with another operation."` Plain `execute`, `executeAsync`, `loadFile`, and `loadFileAsync` **bypass the queue** (call native directly) — they never throw "busy". This is the single biggest behavioral gotcha — see [`references/concurrency.md`](./references/concurrency.md).
3. **The result object has two views of the rows.** `result.results` is a plain `Row[]`. `result.rows` is the TypeORM-style view with `_array`, `length`, and `item(idx)`. Both exist on every `QueryResult`. Plus `rowsAffected`, optional `insertId`, and optional `metadata`.
4. **`transaction` is async-only and auto-manages BEGIN/COMMIT/ROLLBACK.** If the callback resolves, it commits; if it throws, it rolls back. You can also call `tx.commit()` / `tx.rollback()` manually.
5. **You use the JS `open()` wrapper, not the raw native module.** `open({ name })` returns a `NitroSQLiteConnection` bound to that database name, so you never pass the db name into every call. It also registers the database in the operation queue and throws if it's already open.

> **Requires React Native 0.75+ with the New Architecture, plus `react-native-nitro-modules` (>= 0.35.0) as a peer dependency.** (The README mentions 0.71+, but `package.json` pins the peer to RN ≥ 0.75.)

## Routing table — problem to reference

Load the matching file from `references/` before writing code. Each reference cites real APIs verified against the library source.

| User is asking about… | Read |
|---|---|
| Installing, native build config, FTS5/Geopoly flags, system SQLite on iOS, app groups, Expo prebuild, RN/Nitro version requirements | [`references/setup.md`](./references/setup.md) |
| `open()`, `NitroSQLiteConnection`, `location`, opening/loading an existing DB, `close()`, `delete()`, "already open" errors, file locations on iOS/Android | [`references/connections.md`](./references/connections.md) |
| `execute` / `executeAsync`, parameter binding (`?`), the `QueryResult` shape (`results`, `rows._array/length/item`, `rowsAffected`, `insertId`, `metadata`), typed rows, `ColumnType`, storing `null`/blobs | [`references/queries.md`](./references/queries.md) |
| `db.transaction()`, `tx.execute` / `tx.executeAsync` / `tx.commit` / `tx.rollback`, auto commit/rollback, finalized-transaction errors | [`references/transactions.md`](./references/transactions.md) |
| `executeBatch` / `executeBatchAsync`, `BatchQueryCommand`, one query with many param sets, `loadFile` / `loadFileAsync`, `FileLoadResult` | [`references/batch-and-files.md`](./references/batch-and-files.md) |
| Sync vs async, the per-database queue, "Database is busy" errors, ordering guarantees, keeping the UI responsive, `setImmediate` scheduling | [`references/concurrency.md`](./references/concurrency.md) |
| `attach` / `detach`, JOINs across database files, aliases | [`references/attach-detach.md`](./references/attach-detach.md) |
| Using it as a TypeORM driver, `typeORMDriver`, babel alias, `patch-package`, `DataSource` config | [`references/typeorm.md`](./references/typeorm.md) |
| `NitroSQLiteError`, error handling, `instanceof`, migrating from `react-native-quick-sqlite`, breaking changes | [`references/migration-and-errors.md`](./references/migration-and-errors.md) |

If the question doesn't match any row, read [`references/connections.md`](./references/connections.md) first — most usage starts with `open()`.

## Installation

```bash
npm install react-native-nitro-sqlite react-native-nitro-modules
npx pod-install
```

Expo:
```bash
npx expo install react-native-nitro-sqlite react-native-nitro-modules
npx expo prebuild
```

There is **no Expo config plugin** and no manual native linking — Nitro autolinks. Rebuild the app after install. See [`references/setup.md`](./references/setup.md) for compile-time options.

## Quick reference — full API surface

```ts
import { open, NitroSQLiteError, typeORMDriver } from 'react-native-nitro-sqlite'
import type {
  NitroSQLiteConnection,
  QueryResult,
  BatchQueryCommand,
  BatchQueryResult,
  FileLoadResult,
  Transaction,
  SQLiteValue,
  ColumnType,
} from 'react-native-nitro-sqlite'

// Open — returns a connection bound to this db name
const db = open({ name: 'app.sqlite' })          // { name, location? }

// Query (sync — blocks JS thread)
const r = db.execute('SELECT * FROM users WHERE id = ?', [1])
r.results        // Row[]  (plain array of row objects)
r.rows._array    // Row[]  (TypeORM-style view)
r.rows.length    // number
r.rows.item(0)   // Row | undefined
r.rowsAffected   // number
r.insertId       // number | undefined  (after INSERT)
r.metadata       // Record<colName, { name, type: ColumnType, index }> | undefined

// Query (async — runs off the JS thread)
const r2 = await db.executeAsync<{ id: number; name: string }>('SELECT * FROM users')

// Transaction (async only; auto commit / rollback)
await db.transaction(async (tx) => {
  tx.execute('INSERT INTO users (id, name) VALUES (?, ?)', [1, 'Marc'])
  await tx.executeAsync('UPDATE users SET name = ? WHERE id = ?', ['Marc', 1])
  // throw → rollback; resolve → commit; or tx.commit() / tx.rollback()
})

// Batch (one transaction, many statements)
const batch: BatchQueryCommand[] = [
  { query: 'CREATE TABLE IF NOT EXISTS t (id INTEGER, age INTEGER)' },
  { query: 'INSERT INTO t (id, age) VALUES (?, ?)', params: [1, 10] },
  // one query, many param sets:
  { query: 'INSERT INTO t (id, age) VALUES (?, ?)', params: [[2, 20], [3, 30]] },
]
db.executeBatch(batch)            // sync (throws if db busy)
await db.executeBatchAsync(batch) // async (queued)

// Load a .sql file
db.loadFile('/abs/path/dump.sql')           // FileLoadResult { rowsAffected?, commands? }
await db.loadFileAsync('/abs/path/dump.sql')

// Attach / detach another database file
db.attach('other.sqlite', 'other', '/dir')  // (dbNameToAttach, alias, location?)
db.detach('other')

// Lifecycle
db.close()    // close connection (and free the queue)
db.delete()   // drop/delete the database file
```

## SQLite value types

`SQLiteValue = boolean | number | string | ArrayBuffer | null`. Bind these as params and read them back from rows. `ArrayBuffer` maps to SQLite BLOB. `ColumnType` enum: `BOOLEAN`, `NUMBER`, `INT64`, `TEXT`, `ARRAY_BUFFER`, `NULL_VALUE`.

## Testing / verifying the skill is loaded

Good test questions:

- *"How do I run many inserts without blocking the UI?"* → should use `executeAsync` or `executeBatchAsync` / `transaction`, and mention the operation queue — **not** a synchronous loop of `db.execute`.
- *"Why does my `db.executeBatch` throw 'Database is busy'?"* → should explain the per-database queue: a sync op can't run while an async op is in flight; use the async variant or `await` the pending op. See [`references/concurrency.md`](./references/concurrency.md).
- *"How do I read the inserted row id and affected rows?"* → `result.insertId` and `result.rowsAffected`, and rows via `result.results` or `result.rows._array`.
- *"How do I wire this into TypeORM?"* → `typeORMDriver` + babel `module-resolver` alias of `react-native-sqlite-storage` + `patch-package` to expose TypeORM's `package.json`.

## References

| File | Description |
|------|-------------|
| [setup.md](./references/setup.md) | Install, native build config, FTS5/Geopoly, system SQLite, app groups, Expo, version requirements |
| [connections.md](./references/connections.md) | `open()`, connection object, locations, existing databases, close/delete |
| [queries.md](./references/queries.md) | `execute`/`executeAsync`, params, result shape, typed rows, metadata, blobs |
| [transactions.md](./references/transactions.md) | `db.transaction`, the `tx` object, auto/manual commit & rollback |
| [batch-and-files.md](./references/batch-and-files.md) | `executeBatch`/`Async`, `BatchQueryCommand`, `loadFile`/`Async` |
| [concurrency.md](./references/concurrency.md) | Sync vs async, the per-database operation queue, "busy" errors, ordering |
| [attach-detach.md](./references/attach-detach.md) | `attach`/`detach`, cross-database JOINs |
| [typeorm.md](./references/typeorm.md) | TypeORM driver setup and configuration |
| [migration-and-errors.md](./references/migration-and-errors.md) | `NitroSQLiteError`, migrating from `react-native-quick-sqlite` |

## Problem → Reference Mapping

| Problem | Reference | Action |
|---------|-----------|--------|
| Don't know where to start | [connections.md](./references/connections.md) | `open({ name })` |
| App freezes during DB work | [concurrency.md](./references/concurrency.md) | Switch to `executeAsync` / `transaction` |
| "Database is busy" error | [concurrency.md](./references/concurrency.md) | Use async variant or `await` pending op |
| Need atomic multi-statement writes | [transactions.md](./references/transactions.md) | `await db.transaction(...)` |
| Bulk insert many rows | [batch-and-files.md](./references/batch-and-files.md) | `executeBatchAsync` with array-of-arrays params |
| Read insert id / affected rows | [queries.md](./references/queries.md) | `result.insertId` / `result.rowsAffected` |
| JOIN across two db files | [attach-detach.md](./references/attach-detach.md) | `db.attach(file, alias)` |
| Use an ORM | [typeorm.md](./references/typeorm.md) | `typeORMDriver` + babel alias |
| Coming from quick-sqlite | [migration-and-errors.md](./references/migration-and-errors.md) | Rename package, update peer deps |
| Need FTS5 / full-text search | [setup.md](./references/setup.md) | Add `SQLITE_ENABLE_FTS5=1` compile flag |
