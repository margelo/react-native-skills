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

Generated files under `nitrogen/generated/` are outputs. Change the `.nitro.ts` spec or native implementation source, then re-run nitrogen instead of manually editing generated files. These files can be committed to git, and many Nitro libraries do commit them, but the repo policy can choose otherwise. They must be included in the npm package so consumers can build the native library.

## Pair With API Design

Use `api-design` first when shaping the public TypeScript, JavaScript, React, or React Native API. This skill adds the Nitro-specific constraints: HybridObject state, generated specs, native resource ownership, zero-copy data, threading, platform implementation, codegen, and real-device validation.

If the user is building a JS-only React or React Native library, do not apply this skill unless Nitro, HybridObjects, native modules, codegen, C++/Swift/Kotlin bindings, or `react-native-nitro-modules` are part of the task.

## API Freshness

Before choosing APIs, versions, platform requirements, templates, config shape, or native implementation details, verify current sources instead of relying on model memory. Mobile APIs, React Native, Nitro, Gradle, Xcode, Swift, Kotlin, NDK, and package tooling move quickly.

- Prefer official docs, source repositories, release notes, changelogs, package READMEs, and generated templates.
- Look for `llms.txt` or `llms-full.txt` on official docs sites when available, and use those as compact current context.
- Use current package metadata or source when deciding install commands, minimum versions, or config fields.
- Treat remembered API details as a starting hypothesis only. If current docs or source disagree, follow the current docs/source and mention the change when relevant.
- Avoid basing Nitro, native, or example-app decisions on old blog posts, stale snippets, or past trained memory when an official current source is available.

## Repository Defaults

Use these defaults when creating or reorganizing a Nitro Module repo, unless the existing repo or the user explicitly points elsewhere:

- Keep the root clean so `README.md` stays visible in GitHub's file list. The root should mostly be `README.md`, `package.json`, `bun.lock`, `packages/`, one example-app location, optional `docs/`, optional `scripts/`, `config/`, `.github/`, and only config files that tools must discover from the root.
- Prefer Bun wherever possible: `bun install`, `bun run`, `bun --cwd`, and `bunx` for one-off executables. Fall back to `npx` only when Bun cannot run a tool correctly.
- Put publishable libraries in `packages/<package-name>/`. Use multiple packages only when they are independently consumed, versioned, or installed.
- Prefer `apps/<example-name>/` when multiple example apps are needed or likely, including optional native dependencies, feature variants, or separate integration demos. `apps/example` is still a good default even with one app when future examples are plausible. Use top-level `example/` only for small single-example repos where staying close to React Native's generated config is more valuable.
- Keep example apps as close to the official React Native template as possible. Use official APIs, generated configs, and template-supported extension points before custom setup.
- Add `docs/` only when docs are meaningful; prefer Fumadocs for a full docs site.
- Add `scripts/` only for reusable repo automation, not for one-off command notes.
- Put shared config under `config/` when each tool can reliably reference it, such as TypeScript configs, lint/format configs, `.swift-format`, `.clang-format`, and `.editorconfig`. If a tool or editor requires a root config file, keep the root file minimal and point it at `config/`.
- Add `.github/workflows/` for CI validation. Include TypeScript/build checks, JS lint/format with Biome or ESLint/Prettier, and native lint jobs as the codebase matures, such as SwiftLint/SwiftFormat, clang-format/clang-tidy, ktlint, or Detekt.
- Do not add Husky, commitlint, lint-staged, pre-commit hooks, pre-push hooks, or `prepare` scripts that install hooks. Validation belongs in CI; local scripts can exist for manual use but must not block committing or pushing.
- Avoid patches, `patch-package`, postinstall rewrites, monkeypatching, and hacky workarounds unless there is no reasonable alternative. They create maintenance burden, technical debt, and make the project less approachable for new contributors.
- Solve problems at the root cause instead of layering workaround code around symptoms. If a build or example app needs ugly manual plumbing, first revisit the package layout, autolinking, official config APIs, or upstream issue instead of normalizing the workaround.
- Work on a separate branch and open a draft PR early so CI can run while local development continues.
- Use squash merges to keep `main` history clean.
- Once the repo is past the initial release or major rewrite phase, keep PRs atomic and decoupled. Do not cram unrelated features or fixes into one branch unless they are tightly connected.

## Nitro API Design Rules

- Prefer Nitro Modules over TurboModules or handwritten JSI for native module work. Nitro is usually faster and safer because it avoids many raw JSI lifetime, threading, and runtime-destruction hazards. Use raw JSI only when Nitro's Raw JSI Methods are truly required.
- For larger libraries, prefer one small public factory/entry HybridObject that creates stateful domain objects. Keep the root default-constructible for autolinking, and create objects that need arguments through factory methods.
- Design around native state when it improves the API. A `HybridObject` can represent a native resource, prewarmed engine, file, image, database, sensor session, stream, or other stateful object instead of forcing everything into static module functions.
- Use a product/domain noun for the exported JS factory object, not the generated spec type name. For example, export `VisionCamera = createHybridObject<CameraFactory>('CameraFactory')` or `Images = createHybridObject<ImageFactory>('ImageFactory')`. This avoids collisions with `CameraFactory`/`ImageFactory` types without mechanically lowercasing them or adding `Hybrid` prefixes.
- Use factory HybridObjects for resources that require construction arguments, async setup, I/O, or validation. Example: expose `Files.loadFileFromPath(path): Promise<File>` from a `FileFactory` spec instead of trying to construct `File` directly from JS.
- Keep HybridObjects focused on one purpose or lifecycle. Avoid giant native objects that own unrelated domains.
- Autolink only public roots, factories, views, or global utilities that JS must construct directly. Other HybridObjects can be returned from factory methods and do not need their own `nitro.json` autolinking entries.
- For native extension points, pair a JS-facing base HybridObject spec with a public native protocol/interface. The base spec lets JS pass the object through typed APIs; the native protocol/interface exposes platform-specific handles and behavior for first-party and third-party native code.
- When accepting an extensible HybridObject from JS, accept the generated base spec type, then cast to the native protocol/interface on the native side and throw a clear error if it does not conform. This keeps JS portable while native integrations stay strongly typed.
- Use sync methods by default only for fast, in-process work, cheap native object creation, cached metadata, and local transforms. Use `Promise` for permissions, hardware/session setup, I/O, capture/recording, platform async APIs, blocking work, or anything that can take meaningful time.
- As a rule of thumb, benchmark the method and make it async if it takes longer than roughly 50ms.
- If a transform has both cheap and potentially heavy paths, consider explicit sync and async twins such as `convertX()` and `convertXAsync()` instead of hiding blocking work behind one ambiguous method.
- Use properties for cheap observed state or capability, especially readonly capability flags such as `readonly isAccelerometerAvailable: boolean`. Use methods for side effects, expensive work, allocation, mutation, or failure.
- Prefer typed structs, interfaces, string literal unions, readonly properties, and explicit methods over `AnyMap`, `Record<string, unknown>`, stringly typed commands, or loosely shaped event payloads. Use runtime enums only when callers need runtime enum values.
- Use structs for meaningful domain shapes, option groups, and same-type parameter clusters. Do not wrap unrelated hot-path values in a struct only to reduce argument count; Nitro eagerly converts structs, so unnecessary wrappers can be slower than explicit parameters.
- Avoid variants only when a simpler typed model expresses the state. Use variants or discriminated unions when they are the clearest representation, even if they have some runtime overhead.
- Use `ArrayBuffer` for zero-copy native data access. When receiving an `ArrayBuffer` from JS and using it on another thread, copy it first if it is not owning.
- Use `Error` in TypeScript specs for real JS Error prototypes instead of custom typed error objects.
- Use listener functions for repeated events and callbacks, returning a `ListenerSubscription`-style object with `remove()`. Avoid mutable callback properties unless there is a strong reason.
- Use callback option structs for one-shot operation progress when callbacks belong to one method call, such as capture or recording progress callbacks.
- Use `setOn...Callback(callback | undefined)` only for single hot-path callbacks owned by an object, where replacing or removing the callback is the natural operation.
- Use `Sync<(...) => ...>` callbacks only for rare thread-bound hot paths that must synchronously execute on a specific JS runtime or worklet thread.
- Put ergonomic defaults and convenience shaping in TypeScript when that keeps the native Nitro spec simpler and more explicit. For example, normalize optional options in TS before calling a stricter native method.
- Preserve React Native's cross-platform abstraction. Do not expose AVFoundation, Android framework, or platform-specific class names in public APIs unless direct low-level access is the point.
- For Nitro Views, expose the raw `getHostComponent` wrapper plus higher-level React components or hooks when they materially improve ergonomics. Keep them layered over the same native objects and refs.
- Document Nitro public specs heavily with JSDoc. Include lifecycle, defaults, platform availability, performance costs, disposal requirements, and examples directly on exported interfaces.

## Nitro Native Implementation Rules

- Check minimum requirements before debugging weird build failures: Nitro requires React Native 0.75+, Xcode 16.4+, Swift 5.9+, Android compileSdkVersion 34+, and NDK 27+. Nitro Views require React Native 0.78+ and the New Architecture.
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
3. **Example app** — Should an example app be created to test the module, and where should it live? *(Recommended — default: yes; `apps/example` when multiple examples are needed or likely, `example` only for a small single-example repo that should stay close to generated RN config)*
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
# 0. Work on a separate branch; open a draft PR after the first useful commit

# 1. Scaffold
bunx nitrogen@latest init react-native-math

# 2. Run codegen (from package folder after writing spec + nitro.json)
cd packages/react-native-math && bunx nitrogen

# 3. Create example app
bunx @react-native-community/cli@latest init --skip-install MathExample
mkdir -p apps && mv MathExample apps/example
# Alternative: mv MathExample example

# 4. Install and test
cd apps/example && bun add ../../packages/react-native-math
bun add react-native-nitro-modules@<same-version-as-package>
bun example android
bun example ios
```

Full step-by-step references below.

## When to Apply

Reference these guidelines when:
- Creating any new React Native native module using the Nitro framework
- Checking Nitro minimum platform requirements
- Verifying current APIs, templates, dependency versions, and tooling behavior before making implementation decisions
- Designing or reviewing the public API shape of a Nitro-backed library
- Deciding whether an API should be static, instance-based, sync, async, callback-based, or resource-backed
- Writing HybridObject TypeScript specs (`*.nitro.ts` files)
- Running Nitrogen codegen and implementing generated interfaces
- Setting up a monorepo example app for a Nitro library
- Choosing repository layout, root cleanliness, shared config placement, and CI shape
- Establishing branch, draft PR, and squash-merge workflow for a Nitro library
- Configuring Android Gradle paths for a monorepo structure
- Debugging autolinking failures or missing generated files
- Preparing a Nitro module package for npm publishing
- Setting up one-command releases with `release-it` and `bun release`

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
| 11 | npm publish and release prep | MEDIUM | [spec-package-publish.md][spec-package-publish] |
| 12 | VisionCamera-style full library patterns | MEDIUM | [vision-camera-golden-standard.md][vision-camera-golden-standard] |

## Quick Reference

### Minimum HybridObject Spec (`src/specs/Math.nitro.ts`)

```typescript
import type { HybridObject } from 'react-native-nitro-modules'

export interface Math extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  add(a: number, b: number): number
}
```

### Minimum Runtime + Type Exports (`src/index.ts`)

```typescript
import { NitroModules } from 'react-native-nitro-modules'
import type { Math } from './specs/Math.nitro'

export const math = NitroModules.createHybridObject<Math>('Math')
export type { Math } from './specs/Math.nitro'
```

### Minimum `nitro.json`

```json
{
  "$schema": "https://nitro.margelo.com/nitro.schema.json",
  "cxxNamespace": ["math"],
  "ios": { "iosModuleName": "NitroMath" },
  "android": {
    "androidNamespace": ["math"],
    "androidCxxLibName": "NitroMath"
  },
  "autolinking": {
    "Math": {
      "ios": {
        "language": "swift",
        "implementationClassName": "HybridMath"
      },
      "android": {
        "language": "kotlin",
        "implementationClassName": "HybridMath"
      }
    }
  }
}
```

### Root `package.json` Scripts

```json
{
  "scripts": {
    "specs": "bun --cwd packages/react-native-math run specs",
    "example": "bun --cwd apps/example"
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
| [spec-package-publish.md][spec-package-publish] | `package.json` author, `files` field, npm publish readiness, and `release-it` setup |
| [vision-camera-golden-standard.md][vision-camera-golden-standard] | Package layout, API layering, Nitro object modeling, and publishing patterns inspired by VisionCamera |

## Problem → Skill Mapping

| Problem | Reference | Action |
|---------|-----------|--------|
| Need to design the public API first | `api-design` + this SKILL.md | Shape the TS/React API, then apply Nitro constraints |
| Need latest APIs or setup guidance | Current official docs/source + this SKILL.md | Check official docs, release notes, source repos, package metadata, or `llms-full.txt` before deciding |
| Need a recommended repo structure | [setup-monorepo-init.md][setup-monorepo-init] | Use `packages/`, `apps/` or `example/`, optional `docs/`, `scripts/`, `config/`, and `.github/workflows/` |
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
| Need one-command releases | [spec-package-publish.md][spec-package-publish] | Configure `release-it` behind `bun release` |
| Need a full-featured library structure | [vision-camera-golden-standard.md][vision-camera-golden-standard] | Use the VisionCamera-inspired package, API, hooks, views, and Nitro object model |

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
[vision-camera-golden-standard]: references/vision-camera-golden-standard.md
