---
id: setup
title: Installation and native configuration
scope: react-native-nitro-sqlite
keywords: install, pod-install, expo, prebuild, FTS5, Geopoly, compile flags, GCC_PREPROCESSOR_DEFINITIONS, NITRO_SQLITE_USE_PHONE_VERSION, system sqlite, app groups, RNNitroSQLite_AppGroup, nitro-modules, new architecture, peer dependency
---

# Installation and native configuration

## Mental model

`react-native-nitro-sqlite` is a Nitro Module: it autolinks, has **no Expo config plugin**, and requires the **New Architecture**. The native side bundles its own SQLite C build by default (so behavior is consistent across OS versions), but you can opt into the OS's system SQLite or flip SQLite compile-time flags.

## Requirements

- **React Native ≥ 0.75** with the New Architecture enabled. (README text says 0.71+, but `peerDependencies` pins `react-native >= 0.75.0`.)
- **`react-native-nitro-modules` ≥ 0.35.0** — a required peer dependency, installed alongside.
- iOS and Android. (No web.)

## Install

```bash
npm install react-native-nitro-sqlite react-native-nitro-modules
npx pod-install
```

Expo (bare/prebuild — this is not a config-plugin library, so you must prebuild):

```bash
npx expo install react-native-nitro-sqlite react-native-nitro-modules
npx expo prebuild
```

Rebuild the native app after installing. No `Podfile` edits, no `MainApplication` registration — Nitro handles linking.

## Use the system (phone) SQLite on iOS

By default the bundled SQLite is compiled in. To link against the OS's system SQLite instead (smaller binary, but version varies by iOS release):

```bash
NITRO_SQLITE_USE_PHONE_VERSION=1 npx pod-install
```

## Compile-time SQLite options (FTS5, Geopoly, JSON1, etc.)

SQLite features like FTS5 full-text search or Geopoly are enabled with preprocessor defines on the native target.

**iOS** — in your app's `ios/Podfile`, inside a `post_install` block:

```ruby
post_install do |installer|
  installer.pods_project.targets.each do |target|
    if target.name == "RNNitroSQLite"
      target.build_configurations.each do |config|
        config.build_settings['GCC_PREPROCESSOR_DEFINITIONS'] ||= ['$(inherited)']
        config.build_settings['GCC_PREPROCESSOR_DEFINITIONS'] << 'SQLITE_ENABLE_FTS5=1'
        # add more flags by pushing additional entries
      end
    end
  end
end
```

**Android** — in `android/gradle.properties`:

```properties
nitroSqliteFlags="-DSQLITE_ENABLE_FTS5=1"
```

Multiple flags: combine them in the same string, e.g. `nitroSqliteFlags="-DSQLITE_ENABLE_FTS5=1 -DSQLITE_ENABLE_GEOPOLY=1"`.

> Note: enabling FTS5 etc. via flags only works when compiling the **bundled** SQLite (the default). It won't take effect if you switch to the system SQLite on iOS via `NITRO_SQLITE_USE_PHONE_VERSION`.

## iOS App Groups (share a DB with extensions / widgets)

To store the database inside an App Group container (so an app extension can read it):

1. Add the **App Groups** capability in Xcode and create/select your group id (e.g. `group.com.myapp`).
2. In your app's `Info.plist`, set the key `RNNitroSQLite_AppGroup` to that group id.

The database files then live in the shared app-group directory instead of the app's documents directory.

## Gotchas

- **Old Architecture is not supported.** Nitro Modules require the New Architecture; if the app is on the legacy bridge, enable New Arch first.
- **Forgetting `react-native-nitro-modules`.** It is a separate required peer dependency; installing only `react-native-nitro-sqlite` will fail at runtime.
- **Expo Go won't work.** There's native code; use a development build (`expo prebuild` + custom dev client), not Expo Go.
- **Rebuild after install/flag changes.** Compile-flag changes require a clean native rebuild (`pod install` again on iOS; rebuild on Android).

## Pointers

- Nitro Modules: https://nitro.margelo.com/
- Repo: https://github.com/margelo/react-native-nitro-sqlite
- Related: [connections.md](./connections.md)
