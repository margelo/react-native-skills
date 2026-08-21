---
id: connections
title: Opening connections and managing databases
scope: react-native-nitro-sqlite
keywords: open, NitroSQLiteConnection, name, location, close, delete, drop, already open, existing database, documents directory, files directory, sandbox, path
---

# Opening connections and managing databases

## Mental model

You always work through the JS `open()` wrapper, **not** the raw native module. `open({ name })` does three things:

1. Opens (creating if missing) the SQLite file via the native module.
2. Registers the database in the internal **operation queue** (see [concurrency.md](./concurrency.md)).
3. Returns a `NitroSQLiteConnection` whose methods already know the db name — you never repeat it.

```ts
import { open } from 'react-native-nitro-sqlite'
import type { NitroSQLiteConnection } from 'react-native-nitro-sqlite'

let db: NitroSQLiteConnection = open({ name: 'app.sqlite' })
```

## `open(options)`

```ts
interface NitroSQLiteConnectionOptions {
  name: string        // file name (and the key used in the operation queue)
  location?: string   // optional directory; defaults to the app's data dir
}
```

- `name` is the database file name **and** the identity used by the queue and by `attach`/`close`/`delete`.
- `location` lets you open the file in a non-default directory (see "File locations" below).

**Opening the same name twice throws.** The queue rejects a second `open` of an already-open name:

```
NitroSQLiteError: Database app.sqlite is already open. There is already a connection to the database.
```

Keep a single connection per database name (e.g. a module-level singleton) and reuse it.

## The connection object

```ts
interface NitroSQLiteConnection {
  close(): void
  delete(): void
  attach(dbNameToAttach: string, alias: string, location?: string): void
  detach(alias: string): void
  transaction: <R = void>(cb: (tx: Transaction) => Promise<R>) => Promise<R>
  execute: ExecuteQuery                 // sync
  executeAsync: ExecuteAsyncQuery       // async
  executeBatch(commands: BatchQueryCommand[]): BatchQueryResult
  executeBatchAsync(commands: BatchQueryCommand[]): Promise<BatchQueryResult>
  loadFile(location: string): FileLoadResult
  loadFileAsync(location: string): Promise<FileLoadResult>
}
```

Method-by-method: queries → [queries.md](./queries.md); transactions → [transactions.md](./transactions.md); batch & files → [batch-and-files.md](./batch-and-files.md); attach/detach → [attach-detach.md](./attach-detach.md).

## Lifecycle: `close()` and `delete()`

```ts
db.close()    // closes the connection and removes it from the operation queue
db.delete()   // drops/deletes the database file from disk
```

- `close()` frees the queue entry for that name. If there are still queued/in-progress operations, it logs a warning and closes anyway — so `await` your pending async work before closing.
- `delete()` removes the underlying file (it calls the native `drop`). You typically `close()` then `delete()`.
- After `close()`, the **queue-aware** methods (`executeBatch`, `executeBatchAsync`, `transaction`) throw `Database <name> is not open`. Plain `execute`/`executeAsync`/`loadFile` don't perform that JS-level check and will instead fail at the native layer (the handle is gone). Either way, don't use a closed connection — re-`open()` to use it again.

```ts
// Reset pattern
db.close()
db.delete()
db = open({ name: 'app.sqlite' })
```

## File locations

Databases are created under the app's data directory by default:

- **iOS:** the app **Documents** directory (or the App Group container if `RNNitroSQLite_AppGroup` is set — see [setup.md](./setup.md)).
- **Android:** the app **files** directory.

`location` is a directory **relative to that default root**. The native implementation appends it to the root before appending `name`:

```ts
const db = open({ name: 'my.sqlite', location: 'databases/archive' })
```

Relative traversal is also possible inside the sandbox, for example `location: '../www'` for a bundled database at `../www/myDb.sqlite`.

> Do not pass an absolute `location`. It is still appended to the app data root and produces an invalid nested path rather than escaping that root.

> **iOS sandbox:** you cannot access paths outside the app sandbox. To open a DB that lives elsewhere (e.g. a downloaded file), copy/move it into the sandbox first with a file library (`react-native-fs`, `expo-file-system`, etc.), then `open()` it.

## Loading / shipping an existing database

1. Place the `.sqlite` file in your app assets or download it at runtime.
2. Copy it under the app data root.
3. Pass the copied file's directory relative to that root:

```ts
const db = open({
  name: 'prefilled.sqlite',
  location: 'seeded',
})
```

This opens the existing file as-is; no migration is performed.

## Gotchas

- **One connection per name.** A second `open()` of the same name throws "already open".
- **`location` is relative.** Absolute directories are not supported by `open` or `attach`.
- **`delete()` is irreversible.** It removes the file from disk.
- **`close()` before `delete()`** if you intend to reopen, to avoid the busy warning.
- **`name` is the queue key.** Two different files must have two different `name`s; `location` only changes the directory.

## Pointers

- Source: `package/src/operations/session.ts`, `package/src/DatabaseQueue.ts`
- Related: [concurrency.md](./concurrency.md), [attach-detach.md](./attach-detach.md)
