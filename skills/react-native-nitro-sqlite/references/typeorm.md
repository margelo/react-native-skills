---
id: typeorm
title: Using react-native-nitro-sqlite as a TypeORM driver
scope: react-native-nitro-sqlite
keywords: typeorm, typeORMDriver, DataSource, driver, react-native-sqlite-storage, babel, module-resolver, alias, patch-package, package.json exports, reflect-metadata, decorators
---

# Using react-native-nitro-sqlite as a TypeORM driver

## Mental model

The library ships a `typeORMDriver` export that adapts the connection to TypeORM's `react-native` driver contract (callback-based `executeSql`, `transaction`, `close`, `attach`, `detach`). Because of Metro/Node resolution quirks, two extra setup steps are required: exposing TypeORM's `package.json`, and aliasing the driver module name TypeORM expects.

> The `typeORMDriver` object is intended **only** for TypeORM. For normal app code use the `open()` connection API directly.

## Step 1 — Expose TypeORM's `package.json`

TypeORM needs its own `package.json` resolvable. Add this to TypeORM's `package.json` `exports` map:

```json
{
  "exports": {
    "./package.json": "./package.json"
  }
}
```

Persist that change across installs with `patch-package`:

```bash
npx patch-package --exclude 'nothing' typeorm
```

(Make sure `patch-package` runs on `postinstall`.)

## Step 2 — Alias the driver in Babel

TypeORM's React Native driver imports `react-native-sqlite-storage`. Alias it to this library in `babel.config.js`:

```js
module.exports = {
  // ...
  plugins: [
    [
      'module-resolver',
      {
        alias: {
          'react-native-sqlite-storage': 'react-native-nitro-sqlite',
        },
      },
    ],
  ],
}
```

Install the plugin:

```bash
npm i -D babel-plugin-module-resolver
```

You'll also typically need decorator support for TypeORM entities:

```bash
npm i -D babel-plugin-transform-typescript-metadata @babel/plugin-proposal-decorators
```

```js
plugins: [
  'babel-plugin-transform-typescript-metadata',
  ['@babel/plugin-proposal-decorators', { legacy: true }],
  // ...module-resolver above
]
```

And import `reflect-metadata` once at the app entry:

```ts
import 'reflect-metadata'
```

## Step 3 — Configure the DataSource

```ts
import { DataSource } from 'typeorm'
import { typeORMDriver } from 'react-native-nitro-sqlite'

const dataSource = new DataSource({
  type: 'react-native',
  database: 'typeormdb',
  location: '.',
  driver: typeORMDriver,
  entities: [/* your entities */],
  synchronize: true,
})

await dataSource.initialize()
```

- `type: 'react-native'` — TypeORM's RN driver type.
- `driver: typeORMDriver` — the export from this library.
- `location` — directory for the DB file (same meaning as `open`'s `location`).

## What `typeORMDriver` provides

It exposes `openDatabase(options, ok, fail)` returning a connection with `executeSql` (callback-style, backed by `executeAsync`), `transaction`, `close`, `attach`, and `detach`. You don't call these directly — TypeORM does.

## Gotchas

- **Both Babel steps are mandatory.** Missing the alias → TypeORM tries to load `react-native-sqlite-storage` (not installed). Missing decorator plugins → entity decorators fail.
- **`patch-package` must persist.** Without exposing `./package.json`, TypeORM's version detection breaks under Metro.
- **`reflect-metadata` import must come first**, before any entity is imported.
- **Restart Metro with cache reset** after editing `babel.config.js`: `npx react-native start --reset-cache`.

## Pointers

- Source: `package/src/typeORM.ts`
- Related: [connections.md](./connections.md), [setup.md](./setup.md)
