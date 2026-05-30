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

For apps, use a real RN app. Prefer `apps/<name>` when multiple examples are needed or likely, including optional native dependencies or integration variants. A shallower `example/` or standalone example app is valid for intentionally small single-example repos and can keep more generated React Native config working without path rewrites. Use harness/device tests for native behavior.

## Public API Layers

- Export one runtime root factory object, created from the autolinked root HybridObject.
- Name the runtime factory export as a product/domain object, not as the spec type. VisionCamera uses `VisionCamera = createHybridObject<CameraFactory>('CameraFactory')`; Nitro Image uses `Images = createHybridObject<ImageFactory>('ImageFactory')`.
- Keep the generated spec/interface name descriptive, such as `CameraFactory`, and keep the `createHybridObject(...)` key matching `nitro.json`. Only the JS value export gets the product/domain name.
- Export all public specs and common types from `src/index.ts` so users can type advanced integrations.
- Keep specs and public types split by domain. Prefer focused files under `src/specs/common-types/`, `src/specs/outputs/`, `src/specs/instances/`, and similar folders over one large file that contains every option, result, and helper type.
- Layer hooks and React components over the imperative core; do not make hooks the only API.
- Put domain defaults and convenience in hooks/utilities, while keeping Nitro specs explicit.
- Provide native host components for low-level control. Add higher-level React components only when they remove repeated setup code.

## HybridObject Modeling

- Root factory: default-constructible and autolinked, such as `CameraFactory`.
- Resource objects: returned from factory methods, such as sessions, controllers, outputs, recorders, devices, frames, and renderers.
- Capability objects: expose readonly native state and support checks.
- Lifecycle objects: expose `dispose()`, `isValid`, `memorySize`, and cheap observed state when they own native buffers or resources.
- Views: use `HybridView<Props, Methods>` specs, `getHostComponent(...)`, and a `hybridRef` bridge for imperative view methods.
- Native extension points: expose a portable TS base HybridObject spec, then pair it with a public native protocol/interface that can unwrap platform objects. VisionCamera's `CameraOutput` crosses JS/TS, while `NativeCameraOutput` lets native code access `AVCaptureOutput` on iOS or CameraX `UseCase`s on Android.

Autolink only roots, public factories, hybrid views, and global utilities that JS directly creates. Do not autolink every internal object if it is returned by another HybridObject.

Objects returned from factories should be ready for normal use. If construction requires async native setup or validation, make the factory method return `Promise<Thing>` instead of exposing a separate `prepareThing()` step on the returned object.

## Native Extension Points

Use this when first-party or third-party native modules need to plug into the library:

- Define a JS-facing base spec, such as `CameraOutput`, that extends `HybridObject` and contains only portable API.
- Define a public native protocol/interface, such as `NativeCameraOutput`, that exposes platform-native handles and lifecycle hooks.
- Built-in implementations conform to both the generated Nitro spec and the native protocol/interface.
- Session/controller code accepts the generated base spec from JS, casts to the native protocol/interface, and throws a clear error when the object does not conform.
- Place native protocols/interfaces in public source folders, for example `ios/Public` and `android/src/main/java/.../public`, and include them in the npm package so external native modules can conform.

This lets an object remain fully typed and passable from JS/TS while native code can still work with AVFoundation, CameraX, or other platform-specific types without exposing those native concepts in the JS API.

## Sync, Async, And Callbacks

- Use sync properties for cached or immediately observable state.
- Use sync methods for cheap local native object construction, metadata queries, coordinate transforms, and same-thread operations.
- Use `Promise` for permissions, session creation, configuration, start/stop, capture, recording, platform async APIs, I/O, and heavy transforms.
- Provide sync and async variants only when both are genuinely useful, for example a blocking conversion and an async conversion.
- Use `addOn...Listener(...): ListenerSubscription` for repeated events. The subscription owns cleanup through `remove(): void`, should be a flat interface unless it exposes native state beyond cleanup, and must not expose numeric listener IDs or `removeListener(listenerId)` APIs.
- Use callback structs for one-shot operation progress callbacks, such as capture or recording callbacks.
- Use `setOn...Callback(callback | undefined)` for a single replaceable hot-path callback owned by an output object.
- Use `Sync<(...) => ...>` only for thread-bound hot paths where JS must run synchronously on a specific runtime or worklet thread.

## Type And Documentation Style

- Follow `api-design` for exported type shape, naming, string literal unions, optional fields, writable properties, errors, and JSDoc.
- Keep Nitro specs explicit. Put defaults, presets, and convenience adapters in hooks or utilities when that avoids optional native branches.
- Use package and folder structure as namespace. Add prefixes only when the unprefixed type name is ambiguous at package-root import sites.

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
