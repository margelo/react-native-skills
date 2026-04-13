---
title: Preparing the Package for npm Publishing
impact: MEDIUM
tags: package.json, npm, publish, files, author, metadata, npm-pack, podspec
---

# Skill: Preparing the Package for npm Publishing

Covers Step 21: updating `package.json` with correct author info, ensuring all required files are included for npm publishing, and verifying the package is ready for distribution.

## Quick Config

```json
{
  "name": "react-native-math",
  "version": "0.1.0",
  "author": "Your Name <your@email.com>",
  "license": "MIT",
  "files": [
    "src",
    "lib",
    "ios",
    "android",
    "cpp",
    "nitrogen/generated",
    "react-native-math.podspec",
    "nitro.json"
  ]
}
```

## When to Use

- Before publishing to npm for the first time
- When consumers report missing files after installing the package
- When `nitro.json` autolinking fails for consumers of the published package

## Prerequisites

- Module builds and tests pass
- Example app runs on both Android and iOS
- `lib/` directory exists (run the build step first)

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
    "lib",
    "ios",
    "android",
    "cpp",
    "nitrogen/generated",
    "react-native-math.podspec",
    "nitro.json"
  ]
}
```

Critical files that must be included:
- `nitrogen/generated` — Native glue code; consumers need this for builds
- `nitro.json` — Required for autolinking to work
- `react-native-math.podspec` — Required for iOS CocoaPods integration
- `android/` — Android source files and `CMakeLists.txt`
- `ios/` — Swift/ObjC source files
- `cpp/` — C++ implementation files (if using C++)
- `lib/` — Compiled TypeScript output (JS + type definitions)
- `src/` — TypeScript source for consumers who use `react-native` field in package.json

### 3. Verify `main`, `module`, and `types` fields

```json
{
  "main": "lib/commonjs/index.js",
  "module": "lib/module/index.js",
  "types": "lib/typescript/index.d.ts",
  "react-native": "src/index.ts"
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

### 5. Dry run to verify file list

```bash
npm pack --dry-run
```

Check the output — every file listed in step 2 must appear. If `nitro.json` or `nitrogen/generated` are missing, consumers' builds will fail.

### 6. Publish

```bash
npm publish
# or for a scoped package:
npm publish --access public
```

## Code Examples

### Complete `package.json` metadata section

```json
{
  "name": "react-native-math",
  "version": "0.1.0",
  "description": "A fast React Native Math module built with Nitro Modules",
  "main": "lib/commonjs/index.js",
  "module": "lib/module/index.js",
  "types": "lib/typescript/index.d.ts",
  "react-native": "src/index.ts",
  "source": "src/index.ts",
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
  "files": [
    "src",
    "lib",
    "ios",
    "android",
    "cpp",
    "nitrogen/generated",
    "react-native-math.podspec",
    "nitro.json"
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
example/
__tests__/
.github/
*.test.ts
tsconfig.json
babel.config.js
```

## Common Pitfalls

- **Missing `nitrogen/generated` in `files`** — Consumers' native builds will fail because the generated C++ glue code is absent
- **Missing `nitro.json` in `files`** — Autolinking won't work for consumers; they'll get "native module not found" errors
- **Publishing before building** — `lib/` must be populated before publishing; build first
- **Missing `react-native-math.podspec` in `files`** — iOS consumers won't be able to run `pod install`
- **Incorrect `types` path** — Points to a file that doesn't exist after build

## Related Skills

- [setup-monorepo-init.md](setup-monorepo-init.md) — Review the original scaffold structure
- [spec-nitro-json.md](spec-nitro-json.md) — Ensure `nitro.json` is complete before publishing
