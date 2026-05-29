---
title: Preparing the Package for npm Publishing and Releases
impact: MEDIUM
tags: package.json, npm, publish, release-it, files, author, metadata, npm-pack, podspec, changelog, github-release
---

# Skill: Preparing the Package for npm Publishing and Releases

Covers publish metadata, npm package contents, and one-command releases with `release-it`.

## Quick Config

```json
{
  "name": "react-native-math",
  "version": "0.1.0",
  "author": "Your Name <your@email.com>",
  "license": "MIT",
  "scripts": {
    "typecheck": "tsc --noEmit",
    "build": "tsc",
    "specs": "tsc --noEmit false && nitrogen",
    "release": "release-it"
  },
  "files": [
    "src",
    "react-native.config.js",
    "lib",
    "nitrogen",
    "android/build.gradle",
    "android/gradle.properties",
    "android/fix-prefab.gradle",
    "android/CMakeLists.txt",
    "android/src",
    "ios/**/*.h",
    "ios/**/*.m",
    "ios/**/*.mm",
    "ios/**/*.cpp",
    "ios/**/*.swift",
    "cpp/**/*.h",
    "cpp/**/*.hpp",
    "cpp/**/*.cpp",
    "nitro.json",
    "*.podspec",
    "README.md"
  ]
}
```

## When to Use

- Before publishing to npm for the first time
- When consumers report missing files after installing the package
- When `nitro.json` autolinking fails for consumers of the published package
- When setting up `bun release` so maintainers can publish with one command

## Prerequisites

- Module builds and tests pass
- Example app runs on both Android and iOS
- `lib/` directory exists (run the build step first)
- `release-it` is installed and configured before the first public release

## Step-by-Step

### 1. Update author and contact info

In `packages/react-native-math/package.json`:

```json
{
  "author": "Your Full Name <your@email.com>",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/yourusername/react-native-math.git"
  },
  "homepage": "https://github.com/yourusername/react-native-math#readme",
  "bugs": {
    "url": "https://github.com/yourusername/react-native-math/issues"
  }
}
```

### 2. Set the `files` field

This controls what gets uploaded to npm. **Missing files = broken package for consumers.**

```json
{
  "files": [
    "src",
    "react-native.config.js",
    "lib",
    "nitrogen",
    "android/build.gradle",
    "android/gradle.properties",
    "android/fix-prefab.gradle",
    "android/CMakeLists.txt",
    "android/src",
    "ios/**/*.h",
    "ios/**/*.m",
    "ios/**/*.mm",
    "ios/**/*.cpp",
    "ios/**/*.swift",
    "cpp/**/*.h",
    "cpp/**/*.hpp",
    "cpp/**/*.cpp",
    "nitro.json",
    "*.podspec",
    "README.md"
  ]
}
```

Critical files that must be included:
- `nitrogen/` — Generated native glue and autolinking files; consumers need these for builds. These files may or may not be committed to git, but they must be published to npm.
- `nitro.json` — Required for autolinking to work
- `*.podspec` — Required for iOS CocoaPods integration. Prefer a root podspec named after `ios.iosModuleName`, for example `NitroMath.podspec` with `s.name = "NitroMath"`.
- `android/build.gradle`, `android/gradle.properties`, `android/CMakeLists.txt`, and `android/src` — Android build config and source files
- `ios/**/*.{h,m,mm,cpp,swift}` — iOS source files
- `cpp/**/*.{h,hpp,cpp}` — C++ implementation files (if using C++)
- `lib/` — Compiled TypeScript output (JS + type definitions)
- `src/` — TypeScript source for consumers who use `react-native` field in package.json
- `react-native.config.js` — Required if the package customizes React Native autolinking

### 3. Verify `main`, `module`, and `types` fields

```json
{
  "main": "lib/index",
  "module": "lib/index",
  "types": "lib/index.d.ts",
  "react-native": "src/index",
  "source": "src/index"
}
```

### 4. Build the package

```bash
cd packages/react-native-math
bun run build
# or:
bun run prepare
```

Ensure `lib/` is populated before packing.

Run `bun run specs` before publishing whenever `.nitro.ts` files changed so `nitrogen/` is current.

### 5. Dry run to verify file list

```bash
npm pack --dry-run
```

Check the output — every file listed in step 2 must appear. If `nitro.json`, `nitrogen/`, or `*.podspec` are missing, consumers' builds will fail.

### 6. Configure one-command releases with release-it

All releasable Nitro repos should expose one command:

```bash
bun release
```

Use `release-it` for this. For a single-package repo, the package can run `release-it` directly and own npm publishing:

```json
{
  "scripts": {
    "release": "release-it"
  },
  "devDependencies": {
    "@release-it/conventional-changelog": "^11.0.0",
    "release-it": "^20.0.0"
  },
  "release-it": {
    "npm": {
      "publish": true
    },
    "github": {
      "release": true
    },
    "plugins": {
      "@release-it/conventional-changelog": {
        "preset": "conventionalcommits"
      }
    }
  }
}
```

For a monorepo with multiple npm packages, release from the root with `bun release`, but keep responsibilities split:

- Each package has its own `"release": "release-it"` script and package-level `release-it` config with `"npm.publish": true`, `"git": false`, and `"github.release": false`.
- The root `"release": "./scripts/release.sh"` script runs each package release one after another, then runs root `release-it`.
- Root `release-it` owns the version bump commit, tag, changelog, and GitHub release, with `"npm.publish": false`.
- Root `release-it.git.requireCleanWorkingDir` must be `false`; after the first package publishes and bumps files, later package releases and root lockfile updates would otherwise fail the clean-working-tree check.
- After the root version bump, refresh and stage lockfiles before the release commit. Run `bun install` for `bun.lock`, and run the example app's bundle/pod install so `Podfile.lock` is current. Otherwise the next `bun install` or `pod install` produces avoidable uncommitted diffs.

Root script pattern:

```bash
#!/bin/bash
set -e

for pkg in packages/*; do
  [ -d "$pkg" ] || continue
  (cd "$pkg" && bun release "$@")
done

bun run release-it "$@"
```

Root release config shape:

```json
{
  "scripts": {
    "release": "./scripts/release.sh"
  },
  "devDependencies": {
    "@release-it/bumper": "^7.0.0",
    "@release-it/conventional-changelog": "^11.0.0",
    "release-it": "^20.0.0"
  },
  "release-it": {
    "npm": {
      "publish": false
    },
    "git": {
      "commitMessage": "chore: release ${version}",
      "tagName": "v${version}",
      "requireCleanWorkingDir": false
    },
    "github": {
      "release": true
    },
    "hooks": {
      "before:release": "bun install && bun run build && bun specs",
      "after:bump": "bun install && git add bun.lock && bun example bundle-install && bun example pods && git add example/ios/Podfile.lock"
    },
    "plugins": {
      "@release-it/bumper": {
        "out": [
          {
            "file": "packages/react-native-math/package.json",
            "path": "version"
          },
          {
            "file": "example/package.json",
            "path": "version"
          }
        ]
      },
      "@release-it/conventional-changelog": {
        "preset": "conventionalcommits"
      }
    }
  }
}
```

Adapt the lockfile hook to the chosen example layout, such as `apps/example/ios/Podfile.lock` instead of `example/ios/Podfile.lock`. If pod installs are expensive or conditional, put the lockfile refresh in a small `scripts/try-install-lockfiles.sh` helper and call it from `after:bump`.

### 7. Publish

```bash
bun release
```

Do not ask maintainers to remember separate npm publish, tag, changelog, and GitHub release commands. Package npm releases should happen first; the root Git commit, tag, changelog, and GitHub release happen afterward.

## Code Examples

### Complete `package.json` metadata section

```json
{
  "name": "react-native-math",
  "version": "0.1.0",
  "description": "A fast React Native Math module built with Nitro Modules",
  "main": "lib/index",
  "module": "lib/index",
  "types": "lib/index.d.ts",
  "react-native": "src/index",
  "source": "src/index",
  "author": "Your Name <your@email.com>",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/yourusername/react-native-math.git"
  },
  "homepage": "https://github.com/yourusername/react-native-math#readme",
  "bugs": {
    "url": "https://github.com/yourusername/react-native-math/issues"
  },
  "keywords": [
    "react-native",
    "nitro-modules",
    "ios",
    "android"
  ],
  "scripts": {
    "typecheck": "tsc --noEmit",
    "build": "tsc",
    "specs": "tsc --noEmit false && nitrogen",
    "release": "release-it"
  },
  "files": [
    "src",
    "react-native.config.js",
    "lib",
    "nitrogen",
    "android/build.gradle",
    "android/gradle.properties",
    "android/fix-prefab.gradle",
    "android/CMakeLists.txt",
    "android/src",
    "ios/**/*.h",
    "ios/**/*.m",
    "ios/**/*.mm",
    "ios/**/*.cpp",
    "ios/**/*.swift",
    "cpp/**/*.h",
    "cpp/**/*.hpp",
    "cpp/**/*.cpp",
    "nitro.json",
    "*.podspec",
    "README.md"
  ],
  "peerDependencies": {
    "react": "*",
    "react-native": "*",
    "react-native-nitro-modules": "*"
  }
}
```

### `.npmignore` (optional, to exclude dev files)

```
apps/
__tests__/
.github/
*.test.ts
tsconfig.json
babel.config.js
```

## Common Pitfalls

- **Missing `nitrogen` in `files`** — Consumers' native builds will fail because the generated C++ glue and autolinking files are absent
- **Missing `nitro.json` in `files`** — Autolinking won't work for consumers; they'll get "native module not found" errors
- **Publishing before building** — `lib/` must be populated before publishing; build first
- **No one-command release** — Always alias releases to `bun release`; do not leave maintainers with separate publish, tag, changelog, and GitHub release steps
- **Root clean-working-tree checks in monorepos** — Set root `release-it.git.requireCleanWorkingDir` to `false` when package releases run before the root release commit
- **Stale lockfiles after version bumps** — Refresh and stage `bun.lock` and the example app's `Podfile.lock` before root `release-it` creates the release commit
- **Missing `*.podspec` in `files`** — iOS consumers won't be able to run `pod install`. The VisionCamera-style pattern is a root podspec, not one hidden under `ios/`.
- **Incorrect `types` path** — Points to a file that doesn't exist after build. VisionCamera-style packages use `lib/index.d.ts` when `build` emits a flat `lib/index`.

## Related Skills

- [setup-monorepo-init.md](setup-monorepo-init.md) — Review the original scaffold structure
- [spec-nitro-json.md](spec-nitro-json.md) — Ensure `nitro.json` is complete before publishing
