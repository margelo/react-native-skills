---
name: build-nitro-modules
description: Builds and designs React Native Nitro Modules with Nitrogen, HybridObject TypeScript specs, generated native implementations, zero-copy and native-state APIs, Swift/Kotlin/C++ bindings, example apps, and testing. Use when creating a Nitro Module, adding or reviewing HybridObjects, designing Nitro-specific public APIs, implementing native functionality, or setting up the nitrogen codegen pipeline. Pair with api-design for general library API shape.
license: MIT
metadata:
  author: margelo
  tags: react-native, nitro-modules, nitrogen, hybrid-object, api-design, swift, kotlin, c++, monorepo, native-modules, codegen
---

# Build Nitro Modules

## Overview

End-to-end skill for building a React Native Nitro Module: monorepo scaffolding via Nitrogen, TypeScript HybridObject spec authoring, native code generation, platform implementation (C++/Swift/Kotlin), example app wiring, and publish preparation.

Nitro Modules use a codegen pipeline (`nitrogen`) that reads `.nitro.ts` spec files and generates native C++/Swift/Kotlin boilerplate. You then fill in the implementation. This is fundamentally different from old-style turbo modules.

> **NEVER modify any file inside `nitrogen/generated/`.** These files are fully regenerated every time `npx nitrogen` runs — any manual edits will be silently overwritten. Always edit only the `.nitro.ts` spec file, then re-run nitrogen to regenerate.

## Pair With API Design

Use `api-design` first when shaping the public TypeScript, JavaScript, React, or React Native API. This skill adds the Nitro-specific constraints: HybridObject state, generated specs, native resource ownership, zero-copy data, threading, platform implementation, codegen, and real-device validation.

If the user is building a JS-only React or React Native library, do not apply this skill unless Nitro, HybridObjects, native modules, codegen, C++/Swift/Kotlin bindings, or `react-native-nitro-modules` are part of the task.

## Nitro API Design Rules

- Prefer Nitro Modules over TurboModules or handwritten JSI for native module work. Nitro is usually faster and safer because it avoids many raw JSI lifetime, threading, and runtime-destruction hazards. Use raw JSI only when Nitro's Raw JSI Methods are truly required.
- Design around native state when it improves the API. A `HybridObject` can represent a native resource, prewarmed engine, file, image, database, sensor session, stream, or other stateful object instead of forcing everything into static module functions.
- Use factory HybridObjects for resources that require construction arguments, async setup, I/O, or validation. Example: expose `FileFactory.loadFileFromPath(path): Promise<File>` instead of trying to construct `File` directly from JS.
- Keep HybridObjects focused on one purpose or lifecycle. Avoid giant native objects that own unrelated domains.
- Use sync methods by default only for fast, in-process work. Use `Promise` for hardware calls, I/O, async platform APIs, blocking work, or anything that can take meaningful time.
- Use properties for cheap observed state or capability, especially readonly capability flags such as `readonly isAccelerometerAvailable: boolean`. Use methods for side effects, expensive work, allocation, mutation, or failure.
- Prefer typed structs, interfaces, literal unions, enums, readonly properties, and explicit methods over `AnyMap`, `Record<string, unknown>`, stringly typed commands, or loosely shaped event payloads.
- Avoid variants only when a simpler typed model expresses the state. Use variants or discriminated unions when they are the clearest representation, even if they have some runtime overhead.
- Use `ArrayBuffer` for zero-copy native data access. When receiving an `ArrayBuffer` from JS and using it on another thread, copy it first if it is not owning.
- Use `Error` in TypeScript specs for real JS Error prototypes instead of custom typed error objects.
- Use listener functions for repeated events and callbacks, returning an unsubscribe/subscription handle. Avoid mutable callback properties unless there is a strong reason.
- Use `Sync<(...) => ...>` callbacks only for rare cases that must synchronously block the JS thread, such as specific worklet interop.
- Put ergonomic defaults and convenience shaping in TypeScript when that keeps the native Nitro spec simpler and more explicit. For example, normalize optional options in TS before calling a stricter native method.
- Preserve React Native's cross-platform abstraction. Do not expose AVFoundation, Android framework, or platform-specific class names in public APIs unless direct low-level access is the point.

## Nitro Native Implementation Rules

- Every native HybridObject implementation must implement or inherit from the generated `Hybrid*Spec` for that platform. Never implement a standalone native class and expect Nitro to discover it.
- Make native implementation classes `final` by default unless inheritance is genuinely required. This is especially true for Swift and Kotlin HybridObjects.
- Keep top-level native types in separate files. Nest only truly local private helpers.
- Do not silently swallow errors or unsupported platform behavior. Throw, reject a Promise, or emit through an explicit error callback/listener.
- Prefer Nitro/runtime errors or language-native exceptions that surface cleanly to JS. Avoid Objective-C-style `NSError` public paths unless the generated API specifically requires it.
- For Android context access, use `NitroModules.applicationContext` lazily and throw a clear error if it is unavailable.
- Use `Promise.parallel` for thread-pool work and `Promise.async` for async/coroutine-style work according to the native threading model.
- Implement `memorySize` for HybridObjects that own native resources or large allocations so the JS VM can collect them under memory pressure.
- For Nitro Views, implement `prepareForRecycle` when the view owns state that should be reset before reuse.
- Mix C++ with Swift/Kotlin when it lets shared native logic stay cross-platform while platform HybridObjects provide OS-specific services.

## Nitro Testing Rules

- Prefer behavior tests over type-shape tests. Nitrogen already enforces specs at compile time, so tests should cover real feature behavior, inputs, settings, failure paths, and order-of-execution cases.
- Use `react-native-harness` when available for end-to-end testing in a real React Native environment. For native-heavy libraries, prefer real-device or CI device-farm coverage for the API surface that depends on hardware or OS behavior.

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
- Designing or reviewing the public API shape of a Nitro-backed library
- Deciding whether an API should be static, instance-based, sync, async, callback-based, or resource-backed
- Writing HybridObject TypeScript specs (`*.nitro.ts` files)
- Running Nitrogen codegen and implementing generated interfaces
- Setting up a monorepo example app for a Nitro library
- Configuring Android Gradle paths for a monorepo structure
- Debugging autolinking failures or missing generated files
- Preparing a Nitro module package for npm publishing

## Priority-Ordered Guidelines

| Priority | Category | Impact | Reference |
|----------|----------|--------|-----------|
| 0 | General public API shape | CRITICAL | `api-design` |
| 0 | Nitro API constraints | CRITICAL | This SKILL.md |
| 1 | Monorepo scaffold | CRITICAL | [setup-monorepo-init.md][setup-monorepo-init] |
| 2 | HybridObject spec | CRITICAL | [spec-hybrid-object.md][spec-hybrid-object] |
| 3 | nitro.json autolinking | CRITICAL | [spec-nitro-json.md][spec-nitro-json] |
| 4 | Nitrogen codegen | HIGH | [native-nitrogen-codegen.md][native-nitrogen-codegen] |
| 5 | C++ implementation | HIGH | [native-implement-cpp.md][native-implement-cpp] |
| 6 | Kotlin implementation | HIGH | [native-implement-kotlin.md][native-implement-kotlin] |
| 7 | Swift implementation | HIGH | [native-implement-swift.md][native-implement-swift] |
| 8 | Example app setup *(if requested)* | HIGH | [example-app-setup.md][example-app-setup] |
| 9 | Android Gradle paths *(if example app)* | HIGH | [example-android-config.md][example-android-config] |
| 10 | Metro + install + test *(if example app)* | HIGH | [example-metro-install.md][example-metro-install] |
| 11 | npm publish prep | MEDIUM | [spec-package-publish.md][spec-package-publish] |

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
| Need to design the public API first | `api-design` + this SKILL.md | Shape the TS/React API, then apply Nitro constraints |
| Unsure static module vs instance API | This SKILL.md | Prefer HybridObjects for native state, resources, prewarming, and zero-copy data |
| Don't know where to start | [setup-monorepo-init.md][setup-monorepo-init] | Scaffold with `nitrogen init` |
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
