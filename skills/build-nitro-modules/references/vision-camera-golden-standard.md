---
title: VisionCamera Golden Standard Patterns
impact: MEDIUM
tags: vision-camera, package-layout, api-design, factory, hooks, hybrid-view, publishing
---

# Skill: VisionCamera Golden Standard Patterns

Use this when designing a full-featured Nitro library inspired by `mrousavy/react-native-vision-camera`.

## Package Layout

Use a package layout that separates specs, JS ergonomics, native implementations, generated files, and test apps:

```
packages/react-native-math/
├── NitroMath.podspec
├── nitro.json
├── nitrogen/
├── src/
│   ├── index.ts
│   ├── NitroMath.ts               # runtime root factory export
│   ├── specs/
│   │   ├── common-types/
│   │   ├── inputs/
│   │   ├── outputs/
│   │   ├── instances/
│   │   └── views/
│   ├── hooks/
│   ├── views/
│   └── utils/
├── ios/
│   ├── Hybrid Objects/
│   ├── Public/
│   ├── Extensions/
│   └── Utils/
├── android/src/main/java/com/margelo/nitro/math/
│   ├── hybrids/
│   ├── public/
│   ├── extensions/
│   ├── session/
│   ├── utils/
│   └── views/
└── cpp/
```

For apps, use a real RN app. `apps/<name>` is the VisionCamera-style choice for larger monorepos, but a shallower `example/` or standalone example app is valid and can keep more generated React Native config working without path rewrites. Use harness/device tests for native behavior.

## Public API Layers

- Export one runtime root factory object, created from the autolinked root HybridObject.
- Export all public specs and common types from `src/index.ts` so users can type advanced integrations.
- Layer hooks and React components over the imperative core; do not make hooks the only API.
- Put domain defaults and convenience in hooks/utilities, while keeping Nitro specs explicit.
- Provide native host components for low-level control, then wrap them in higher-level React components when composition improves ergonomics.

## HybridObject Modeling

- Root factory: default-constructible and autolinked, such as `CameraFactory`.
- Resource objects: returned from factory methods, such as sessions, controllers, outputs, recorders, devices, frames, and renderers.
- Capability objects: expose readonly native state and support checks.
- Lifecycle objects: expose `dispose()`, `isValid`, `memorySize`, and cheap observed state when they own native buffers or resources.
- Views: use `HybridView<Props, Methods>` specs, `getHostComponent(...)`, and a `hybridRef` bridge for imperative view methods.

Autolink only roots, public factories, hybrid views, and global utilities that JS directly creates. Do not autolink every internal object if it is returned by another HybridObject.

## Sync, Async, And Callbacks

- Use sync properties for cached or immediately observable state.
- Use sync methods for cheap local native object construction, metadata queries, coordinate transforms, and same-thread operations.
- Use `Promise` for permissions, session creation, configuration, start/stop, capture, recording, platform async APIs, I/O, and heavy transforms.
- Provide sync and async variants only when both are genuinely useful, for example a blocking conversion and an async conversion.
- Use `addOn...Listener(...): ListenerSubscription` for repeated events.
- Use callback structs for one-shot operation progress callbacks, such as capture or recording callbacks.
- Use `setOn...Callback(callback | undefined)` for a single replaceable hot-path callback owned by an output object.
- Use `Sync<(...) => ...>` only for thread-bound hot paths where JS must run synchronously on a specific runtime or worklet thread.

## Type And Documentation Style

- Use `interface` for structs/options and public object shapes.
- Use string literal unions for domain values; avoid TypeScript runtime `enum`s unless a runtime value is required.
- Use `readonly` for observed native state and mutable properties only for direct configuration knobs.
- Use optional fields for defaults. Use `undefined` as "not provided"; reserve `null` for explicit "none".
- Use structural presets with `as const satisfies Record<string, Type>` for common values while allowing user-defined values.
- Use `Error` for JS-facing errors in callback and listener signatures.
- Heavily document exported specs, hooks, components, and constants with JSDoc. Include `@default`, `@throws`, `@platform`, `@example`, `@see`, and performance/lifecycle notes where relevant.

## Publishing Pattern

Use VisionCamera-style package metadata:

```json
{
  "main": "lib/index",
  "module": "lib/index",
  "types": "lib/index.d.ts",
  "react-native": "src/index",
  "source": "src/index",
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
  "scripts": {
    "typecheck": "tsc --noEmit",
    "build": "tsc",
    "specs": "tsc --noEmit false && nitrogen"
  }
}
```

Keep npm package naming separate from native module naming. For example, the npm package can be `react-native-math`, while the root podspec and native module name are `NitroMath.podspec` and `s.name = "NitroMath"`, matching `nitro.json`'s `ios.iosModuleName` and `android.androidCxxLibName`.
