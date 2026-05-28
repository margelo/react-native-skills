---
name: build-nitro-modules
description: Builds React Native Nitro Modules from scratch in a monorepo. Scaffolds with Nitrogen, authors HybridObject TypeScript specs, generates native boilerplate, implements in C++/Swift/Kotlin, wires an example app, and prepares for npm publishing. Use when creating a new Nitro Module, implementing native functionality via HybridObjects, or setting up the nitrogen codegen pipeline.
license: MIT
metadata:
  author: margelo
  tags: react-native, nitro-modules, nitrogen, hybrid-object, swift, kotlin, c++, monorepo, native-modules, codegen
---

# Build Nitro Modules

## Overview

End-to-end skill for building a React Native Nitro Module: monorepo scaffolding via Nitrogen, TypeScript HybridObject spec authoring, native code generation, platform implementation (C++/Swift/Kotlin), example app wiring, and publish preparation.

Nitro Modules use a codegen pipeline (`nitrogen`) that reads `.nitro.ts` spec files and generates native C++/Swift/Kotlin boilerplate. You then fill in the implementation. This is fundamentally different from old-style turbo modules.

> **NEVER modify any file inside `nitrogen/generated/`.** These files are fully regenerated every time `npx nitrogen` runs — any manual edits will be silently overwritten. Always edit only the `.nitro.ts` spec file, then re-run nitrogen to regenerate.

## Ask First — Before Doing Anything

**First, determine what the user wants to do:**

> "Are you creating a **new Nitro Module library** from scratch, or adding a **new HybridObject** to an existing library?"

---

### If creating a new library — ask all of these before any command:

1. **Library name** — What should the library be called? (e.g. `react-native-math`)
2. **Monorepo with `packages/` folder** — Should the library live in `packages/<name>` inside a monorepo? *(Strongly recommended — default: yes)*
3. **Example app** — Should an example app be created to test the module? *(Recommended — default: yes)*
4. **Native languages** — Which platforms and languages?
   - iOS: `swift` (default) or `cpp`
   - Android: `kotlin` (default) or `cpp`
   - Cross-platform C++ only: both `cpp`
5. **Module purpose** — Briefly describe what the module does so the correct spec methods can be designed

Do not proceed past Step 1 of the build sequence until all five questions are answered.

### If adding a HybridObject to an existing library — ask only:

1. **HybridObject name** — What should the new HybridObject be called? (e.g. `Camera`, `Crypto`)
2. **Native languages** — iOS: `swift` or `cpp`? Android: `kotlin` or `cpp`?
3. **Purpose** — What does this HybridObject do?

Then skip directly to [spec-hybrid-object.md][spec-hybrid-object] (write the spec), [spec-nitro-json.md][spec-nitro-json] (add autolinking entry), [native-nitrogen-codegen.md][native-nitrogen-codegen] (re-run nitrogen), and the relevant native implementation file. Skip all setup, monorepo, and example app steps.

## Typical Build Sequence

```bash
# 1. Scaffold
npx nitrogen@latest init react-native-math

# 2. Run codegen (from package folder after writing spec + nitro.json)
cd packages/react-native-math && npx nitrogen

# 3. Create example app
npx @react-native-community/cli@latest init --skip-install MathExample

# 4. Install and test
cd example && bun add ../packages/react-native-math
bun add react-native-nitro-modules
bun example android
bun example ios
```

Full step-by-step references below.

## When to Apply

Reference these guidelines when:
- Creating any new React Native native module using the Nitro framework
- Writing HybridObject TypeScript specs (`*.nitro.ts` files)
- Running Nitrogen codegen and implementing generated interfaces
- Setting up a monorepo example app for a Nitro library
- Configuring Android Gradle paths for a monorepo structure
- Debugging autolinking failures or missing generated files
- Preparing a Nitro module package for npm publishing

## Priority-Ordered Guidelines

| Priority | Category | Impact | Reference |
|----------|----------|--------|-----------|
| 1 | Monorepo scaffold | CRITICAL | [setup-monorepo-init.md][setup-monorepo-init] |
| 2 | API design best practices | CRITICAL | [api-design-best-practices.md][api-design-best-practices] |
| 3 | HybridObject spec | CRITICAL | [spec-hybrid-object.md][spec-hybrid-object] |
| 4 | nitro.json autolinking | CRITICAL | [spec-nitro-json.md][spec-nitro-json] |
| 5 | Nitrogen codegen | HIGH | [native-nitrogen-codegen.md][native-nitrogen-codegen] |
| 6 | C++ implementation | HIGH | [native-implement-cpp.md][native-implement-cpp] |
| 7 | Kotlin implementation | HIGH | [native-implement-kotlin.md][native-implement-kotlin] |
| 8 | Swift implementation | HIGH | [native-implement-swift.md][native-implement-swift] |
| 9 | Example app setup *(if requested)* | HIGH | [example-app-setup.md][example-app-setup] |
| 10 | Android Gradle paths *(if example app)* | HIGH | [example-android-config.md][example-android-config] |
| 11 | Metro + install + test *(if example app)* | HIGH | [example-metro-install.md][example-metro-install] |
| 12 | npm publish prep | MEDIUM | [spec-package-publish.md][spec-package-publish] |

## Quick Reference

### Minimum HybridObject Spec (`src/specs/Math.nitro.ts`)

```typescript
import { type HybridObject, NitroModules } from 'react-native-nitro-modules'

interface Math extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  add(a: number, b: number): number
}

const math = NitroModules.createHybridObject<Math>('Math')
export { math }
```

### Minimum `nitro.json`

```json
{
  "$schema": "https://nitro.margelo.com/nitro.schema.json",
  "cxxNamespace": ["math"],
  "ios": { "iosModuleName": "ReactNativeMath" },
  "android": {
    "androidNamespace": ["math"],
    "androidCxxLibName": "ReactNativeMath"
  },
  "autolinking": {
    "Math": { "swift": "HybridMath", "kotlin": "HybridMath" }
  }
}
```

### Root `package.json` Scripts

```json
{
  "scripts": {
    "specs": "bun --cwd packages/react-native-math run specs",
    "example": "bun --cwd example"
  }
}
```

Run: `bun example android`, `bun example ios`, `bun specs`

## References

| File | Description |
|------|-------------|
| [setup-monorepo-init.md][setup-monorepo-init] | Monorepo workspace structure and `nitrogen init` scaffold |
| [api-design-best-practices.md][api-design-best-practices] | Nitro API shape, typed specs, errors, native state, memory, buffers, hooks, and Harness tests |
| [spec-hybrid-object.md][spec-hybrid-object] | Writing `*.nitro.ts` specs and exporting HybridObjects |
| [spec-nitro-json.md][spec-nitro-json] | `nitro.json` all fields, autolinking, namespace configuration |
| [native-nitrogen-codegen.md][native-nitrogen-codegen] | Running Nitrogen and verifying generated files |
| [native-implement-cpp.md][native-implement-cpp] | Implementing HybridObjects in C++ |
| [native-implement-kotlin.md][native-implement-kotlin] | Implementing HybridObjects in Kotlin (Android) |
| [native-implement-swift.md][native-implement-swift] | Implementing HybridObjects in Swift (iOS) |
| [example-app-setup.md][example-app-setup] | RN CLI example app init, workspace wiring, version alignment |
| [example-android-config.md][example-android-config] | `settings.gradle` and `build.gradle` monorepo path fixes |
| [example-metro-install.md][example-metro-install] | Metro watchFolders, library install, App.tsx usage, test runs |
| [spec-package-publish.md][spec-package-publish] | `package.json` author, `files` field, npm publish readiness |

## Problem → Skill Mapping

| Problem | Reference | Action |
|---------|-----------|--------|
| Don't know where to start | [setup-monorepo-init.md][setup-monorepo-init] | Scaffold with `nitrogen init` |
| API shape unclear | [api-design-best-practices.md][api-design-best-practices] | Prefer typed, instance-based APIs with explicit errors |
| Spec file syntax error | [spec-hybrid-object.md][spec-hybrid-object] | Fix `*.nitro.ts` interface |
| Autolinking not working | [spec-nitro-json.md][spec-nitro-json] | Check `nitro.json` autolinking block |
| Nitrogen generates no files | [native-nitrogen-codegen.md][native-nitrogen-codegen] | Verify spec file extension and run command from right dir |
| C++ types unclear | [native-implement-cpp.md][native-implement-cpp] | Follow type reference links to canonical examples |
| Kotlin compilation error | [native-implement-kotlin.md][native-implement-kotlin] | Check annotations and `override` modifiers |
| Swift compilation error | [native-implement-swift.md][native-implement-swift] | Check class inheritance and property signatures |
| Example app won't build (Android) | [example-android-config.md][example-android-config] | Fix Gradle monorepo path configuration |
| Metro can't resolve library | [example-metro-install.md][example-metro-install] | Add `watchFolders` to `metro.config.js` |
| Version mismatch between example and package | [example-app-setup.md][example-app-setup] | Align `react-native` versions across workspaces |
| Package missing files on npm | [spec-package-publish.md][spec-package-publish] | Fix `files` field in `package.json` |

[setup-monorepo-init]: references/setup-monorepo-init.md
[api-design-best-practices]: references/api-design-best-practices.md
[spec-hybrid-object]: references/spec-hybrid-object.md
[spec-nitro-json]: references/spec-nitro-json.md
[native-nitrogen-codegen]: references/native-nitrogen-codegen.md
[native-implement-cpp]: references/native-implement-cpp.md
[native-implement-kotlin]: references/native-implement-kotlin.md
[native-implement-swift]: references/native-implement-swift.md
[example-app-setup]: references/example-app-setup.md
[example-android-config]: references/example-android-config.md
[example-metro-install]: references/example-metro-install.md
[spec-package-publish]: references/spec-package-publish.md
