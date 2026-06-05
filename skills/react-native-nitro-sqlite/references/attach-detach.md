---
id: attach-detach
title: Attaching and detaching databases
scope: react-native-nitro-sqlite
keywords: attach, detach, alias, cross-database join, multiple databases, ATTACH DATABASE, location
---

# Attaching and detaching databases

## Mental model

SQLite can attach a second database file to an open connection under an **alias**, so you can query both in one statement (e.g. JOIN across files, or copy data between databases). `react-native-nitro-sqlite` exposes this as `attach` / `detach` on the connection.

```ts
const db = open({ name: 'main.sqlite' })

db.attach('stats.sqlite', 'stats', '/optional/dir')
const { results } = db.execute(`
  SELECT u.id, u.name, s.score
  FROM main.users u
  INNER JOIN stats.scores s ON s.user_id = u.id
`)
db.detach('stats')
```

## Signatures

```ts
db.attach(dbNameToAttach: string, alias: string, location?: string): void
db.detach(alias: string): void
```

- `dbNameToAttach` — the file name of the database to attach.
- `alias` — the schema name you reference it by in SQL (`alias.tablename`).
- `location` — optional directory of the file to attach (same semantics as `open`'s `location`; see [connections.md](./connections.md)). Omit to use the default data directory.

Reference tables by alias: the main DB is `main.<table>`, the attached one is `<alias>.<table>`.

## Detaching

Call `detach(alias)` when you no longer need the attached database. Detaching frees the lock/handle. You don't strictly have to detach before closing — **closing the main connection detaches everything** automatically — but detach explicitly if the attach was only needed for a short operation.

## Typical uses

- **Cross-file JOINs** — combine a read-only reference DB with the app's writable DB.
- **Copying / migrating data** — `INSERT INTO main.t SELECT * FROM other.t`.
- **Separating concerns** — keep large/optional datasets in their own file, attach on demand.

```ts
// Copy rows from an attached seed database into the main one
db.attach('seed.sqlite', 'seed')
db.execute('INSERT INTO main.products SELECT * FROM seed.products')
db.detach('seed')
```

## Gotchas

- **`attach`/`detach` are synchronous** (`void`). Wrap heavy cross-db work that follows in `executeAsync`/`transaction` as usual.
- **Alias collisions** — don't reuse an alias that's already attached; detach first.
- **iOS sandbox** applies to the attached file's `location` too — it must be inside the sandbox. Copy external files in first.
- **Attached DBs and transactions** — a single transaction can span main + attached tables since they share the connection.

## Pointers

- Source: `package/src/operations/session.ts` (delegates to native `attach`/`detach`)
- Related: [connections.md](./connections.md), [queries.md](./queries.md)
