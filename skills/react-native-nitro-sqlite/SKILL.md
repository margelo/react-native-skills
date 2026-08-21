---
name: react-native-nitro-sqlite
description: Guides developers using the public react-native-nitro-sqlite API for connections, queries, transactions, batches, migrations, and TypeORM. Use when a React Native project uses react-native-nitro-sqlite or migrates from react-native-quick-sqlite.
license: MIT
metadata:
  author: margelo
  scope: react-native-nitro-sqlite
  tags: react-native, sqlite, database, nitro-modules, transactions, typeorm, migration
---

# react-native-nitro-sqlite

Use this skill as a router. Read only the reference that matches the task before writing code.

## Quick start

```ts
import { open } from 'react-native-nitro-sqlite'
import type { QueryResultRow } from 'react-native-nitro-sqlite'

type UserRow = QueryResultRow & {
  id: number
  name: string
}

const db = open({ name: 'app.sqlite' })

await db.executeAsync(`
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL
  )
`)

const insert = await db.executeAsync(
  'INSERT INTO users (name) VALUES (?)',
  ['Marc'],
)
if (insert.insertId === undefined) {
  throw new Error('Insert did not return a row id')
}

const { results } = await db.executeAsync<UserRow>(
  'SELECT id, name FROM users WHERE id = ?',
  [insert.insertId],
)
```

## Operating rules

- Prefer the connection returned by `open({ name, location? })`; reuse one connection per database name.
- Bind values through `?` parameters. Never interpolate user input into SQL.
- Prefer async APIs for work that may block the JS thread.
- Use `transaction` for dependent atomic work and `executeBatchAsync` for fixed bulk work.
- Consume `rowsAffected` and `insertId` only from the write result that produced them.
- Treat row generics as compile-time assertions; validate untrusted data at runtime.
- Keep schema changes in explicit, versioned migrations.

## References

| Task | Read |
|---|---|
| Installation, native options, FTS5/Geopoly, system SQLite, app groups, Expo | [setup.md](./references/setup.md) |
| Open, reuse, close, delete, locations, existing database files | [connections.md](./references/connections.md) |
| Execute SQL, bind parameters, type rows, inspect results, blobs and booleans | [queries.md](./references/queries.md) |
| Run dependent work atomically with automatic commit or rollback | [transactions.md](./references/transactions.md) |
| Run batches or load constrained one-command-per-line SQL files | [batch-and-files.md](./references/batch-and-files.md) |
| Choose sync/async APIs and understand batch/transaction serialization | [concurrency.md](./references/concurrency.md) |
| Attach another database and query across schemas | [attach-detach.md](./references/attach-detach.md) |
| Configure the TypeORM adapter safely | [typeorm.md](./references/typeorm.md) |
| Handle errors or migrate from `react-native-quick-sqlite` v8 | [migration-and-errors.md](./references/migration-and-errors.md) |
