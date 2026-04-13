---
title: Creating and Wiring the Example App
impact: HIGH
tags: example-app, react-native-cli, workspace, version-alignment, monorepo, init, skip-install
---

# Skill: Creating and Wiring the Example App

Covers Steps 11–13: creating the React Native example app with RN CLI, adding it to the monorepo workspace, and aligning dependency versions.

## Quick Commands

```bash
# Create example app (from monorepo root)
npx @react-native-community/cli@latest init --skip-install MathExample

# Move to example/ folder
mv MathExample example

# Add to workspace, then install
bun install
```

## When to Use

- **Only proceed with this file if the user confirmed they want an example app** (asked in the initial questions)
- After native implementation is complete and you need a testable example app
- When setting up the monorepo for the first time

## Prerequisites

- Library package in `packages/<name>` is scaffolded and implemented
- Root `package.json` has `workspaces` field

## Step-by-Step

### 1. Create the example app

Run from the **monorepo root**:

```bash
npx @react-native-community/cli@latest init --skip-install MathExample
```

- Use `--skip-install` to avoid installing into the wrong directory
- Name the app based on the library (e.g. `MathExample`, `CameraExample`)
- See [RN CLI docs](https://github.com/react-native-community/cli/blob/main/docs/commands.md#init) for additional options

### 2. Move to `example/` folder

The scaffold creates a folder named `MathExample`. Rename it:

```bash
mv MathExample example
```

The monorepo should look like:

```
.
├── packages/
│   └── react-native-math/
├── example/              ← example app lives here
│   ├── android/
│   ├── ios/
│   ├── App.tsx
│   └── package.json
└── package.json          ← root workspace
```

### 3. Add `example` to root workspaces

In the root `package.json`:

```json
{
  "workspaces": [
    "packages/*",
    "example"
  ]
}
```

### 4. Align React Native versions

**This is critical** — two different versions of `react-native` in the same monorepo causes cryptic build failures.

Check the example app's RN version:
```bash
cat example/package.json | grep '"react-native"'
```

Open `packages/react-native-math/package.json` and ensure:
```json
{
  "peerDependencies": {
    "react": "*",
    "react-native": "*",
    "react-native-nitro-modules": "*"
  },
  "devDependencies": {
    "react": "18.3.1",
    "react-native": "0.76.5"
  }
}
```

- If the package's `devDependencies` version is **lower** than the example, **upgrade it** to match
- Also align: `react`, `@babel/core`, `metro`, `@react-native/metro-config`
- There must be **zero duplicate versions** of any shared library

### 5. Install from root

```bash
bun install
```

## Code Examples

### Root `package.json` (after setup)

```json
{
  "name": "react-native-math-root",
  "private": true,
  "workspaces": [
    "packages/*",
    "example"
  ],
  "scripts": {
    "specs": "bun --cwd packages/react-native-math run specs",
    "example": "bun --cwd example"
  }
}
```

### Package `package.json` aligned versions

```json
{
  "name": "react-native-math",
  "peerDependencies": {
    "react": "*",
    "react-native": "*",
    "react-native-nitro-modules": "^0.20.0"
  },
  "devDependencies": {
    "react": "18.3.1",
    "react-native": "0.76.5",
    "react-native-nitro-modules": "^0.20.0"
  }
}
```

### Version check command

```bash
# Check for duplicate react-native installs
find . -name "package.json" -not -path "*/node_modules/*" | xargs grep '"react-native"' | grep -v workspace
```

## Common Pitfalls

- **Forgetting `--skip-install`** — Without it, npm/yarn installs from the wrong directory; use `--skip-install` then `bun install` from root
- **Two RN versions** — Even a minor version mismatch causes cryptic `Invariant Violation` errors at runtime
- **Not moving to `example/`** — The app folder name must match what's in `workspaces`
- **Running `pod install` before workspace is set up** — Do `bun install` from root first, then `pod install`

## Related Skills

- [example-android-config.md](example-android-config.md) — Next: fix Android Gradle paths for monorepo
- [example-metro-install.md](example-metro-install.md) — Next: configure Metro and install the library
