---
title: Android Gradle Configuration for Monorepo
impact: HIGH
tags: android, gradle, settings.gradle, build.gradle, monorepo, paths, hermesCommand, reactNativeDir
---

# Skill: Android Gradle Configuration for Monorepo

Covers Steps 14–15: fixing `settings.gradle` and `app/build.gradle` to point to the correct node_modules in a monorepo.

## Quick Config

Two files need path corrections when the example app is inside a subdirectory of the monorepo:

**`example/android/settings.gradle`** — fix includeBuild path:
```groovy
includeBuild("../../node_modules/@react-native/gradle-plugin")
```

**`example/android/app/build.gradle`** — fix react{} block paths:
```groovy
react {
    reactNativeDir = file("../../../node_modules/react-native")
    codegenDir = file("../../../node_modules/@react-native/codegen")
    cliFile = file("../../../node_modules/react-native/cli.js")
    hermesCommand = "$rootDir/../../node_modules/hermes-compiler/hermesc/%OS-BIN%/hermesc"
    autolinkLibrariesWithApp()
}
```

## When to Use

- Every time an example app is created inside a monorepo
- Whenever Android builds fail with `FileNotFoundException` for gradle-plugin or react-native paths
- Default paths created by `npx react-native init` assume the example is at the monorepo root — they are always wrong in a monorepo

## Prerequisites

- Example app created and moved to `example/` folder
- Root `node_modules/` installed at the monorepo root

## Path Depth Reference

For a monorepo structured as:
```
<root>/                        ← node_modules/ lives here
  example/
    android/
      settings.gradle          ← 2 levels up to root: ../../
      app/
        build.gradle           ← 3 levels up to root: ../../../
```

`$rootDir` in Gradle refers to `<root>/example/android/`.
So `$rootDir/../..` reaches the monorepo root.

## Step-by-Step

### 1. Fix `example/android/settings.gradle`

Open `example/android/settings.gradle` and find:

```groovy
// ❌ Wrong — points to example/node_modules (doesn't exist in monorepo)
includeBuild("../node_modules/@react-native/gradle-plugin")
```

Change to:

```groovy
// ✅ Correct — points to root node_modules
includeBuild("../../node_modules/@react-native/gradle-plugin")
```

### 2. Fix `example/android/app/build.gradle`

Open `example/android/app/build.gradle` and find the `react { }` block. Replace with:

```groovy
react {
    /* Folders */
    // Root of your project (where root package.json lives)
    // root = file("../../")

    // react-native NPM package location
    reactNativeDir = file("../../../node_modules/react-native")

    // @react-native/codegen package location
    codegenDir = file("../../../node_modules/@react-native/codegen")

    // react-native CLI entrypoint
    cliFile = file("../../../node_modules/react-native/cli.js")

    /* Hermes */
    // hermesc is 2 levels up from $rootDir (which is example/android/)
    hermesCommand = "$rootDir/../../node_modules/hermes-compiler/hermesc/%OS-BIN%/hermesc"

    /* Autolinking — keep this */
    autolinkLibrariesWithApp()
}
```

### 3. Verify the paths exist

```bash
# Verify react-native is at root
ls ../../node_modules/react-native/package.json

# Verify codegen
ls ../../node_modules/@react-native/codegen/package.json

# Verify hermes-compiler
ls ../../node_modules/hermes-compiler/
```

Run from `example/android/` to match the relative path perspective.

### 4. Build to verify

```bash
cd example
bun android
# or from root:
bun example android
```

## Code Examples

### Complete `settings.gradle` (corrected)

```groovy
pluginManagement {
    includeBuild("../../node_modules/@react-native/gradle-plugin")  // ← fixed
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

rootProject.name = 'MathExample'
apply from: file("../../node_modules/@react-native-community/cli-platform-android/native_modules.gradle")
applyNativeModulesSettingsGradle(settings)
include ':app'
includeBuild('../../node_modules/@react-native/gradle-plugin')
```

### Complete corrected `react {}` block in `build.gradle`

```groovy
react {
    reactNativeDir = file("../../../node_modules/react-native")
    codegenDir = file("../../../node_modules/@react-native/codegen")
    cliFile = file("../../../node_modules/react-native/cli.js")
    hermesCommand = "$rootDir/../../node_modules/hermes-compiler/hermesc/%OS-BIN%/hermesc"
    autolinkLibrariesWithApp()
}
```

## Common Pitfalls

- **Wrong path depth** — Count the directory levels from the file to the monorepo root carefully
- **`$rootDir` is not the monorepo root** — In Gradle, `$rootDir` is `example/android/`, not the monorepo root
- **Forgetting `autolinkLibrariesWithApp()`** — This must stay in the `react {}` block for autolinking to work
- **Editing both files** — Both `settings.gradle` AND `app/build.gradle` need to be updated; missing one causes different failures
- **Hermes path uses `$rootDir` as base** — `$rootDir/../../node_modules/hermes-compiler/...` goes from `example/android/` up 2 levels to reach monorepo root

## Related Skills

- [example-app-setup.md](example-app-setup.md) — Create the example app first
- [example-metro-install.md](example-metro-install.md) — Next: configure Metro and run the example
