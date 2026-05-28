---
title: Nitro API Design Best Practices
impact: CRITICAL
tags: api-design, best-practices, hybrid-object, types, errors, performance, harness
---

# Skill: Nitro API Design Best Practices

Use this before writing the `.nitro.ts` spec and keep using it while implementing Swift/Kotlin/C++. Nitro benefits most from explicit, typed, instance-based APIs that expose native state safely instead of copying everything through JS.

## Hard Rules

- **Do not inherit from `_base`.** Implement the generated spec directly (`HybridFooSpec`) or the real Nitro base type required by the current API. Generated `_base` types are implementation details and should not be part of library code.
- **Don't create functions that could be simple property getters.** For example, use `readonly isAccelerometerAvailable: boolean` instead of `isAccelerometerAvailable(): boolean` when the value is state-like and cheap to read.
- **Use `is*`, `has*`, or similar prefixes for boolean properties.** Bad: `accelerometerAvailable`. Good: `isAccelerometerAvailable`.
- **Use objects / structs instead of 3+ params.** If a method has three or more related arguments, define a typed options/result struct in the `.nitro.ts` spec.
- **Always prefer typed structs/methods/types over `AnyMap` or `Record<K, V>`.** Nitro benefits from statically typed bindings for better performance. Variants (`A | B`) also require runtime overhead, but those are fine; if they can be avoided it is good, but otherwise its no biggie.
- **Use Nitro's recommended types for TypeScript specs.** Prefer primitives (`number`, `string`, `UInt64`, `boolean`), `ArrayBuffer`, `Promise`, typed structs, and callbacks. Use `Error` instead of custom typed errors, as only those are true JS `Error` prototypes.
- **Do not create multiple types in a single file at top level.** Create multiple files for stuff like this. The only exception is truly local types/structs/classes/enums, which can be nested private.
- **Make almost all classes `final`.** This is better for performance, unless you really expect them to be overridden. Usually that never happens in Nitro HybridObjects.

## Error Handling

- **Avoid swallowing errors silently.** Don't just early return, don't just log/print. Either throw or reject promise if possible, or have a separate error listener if an error is being sent in a different execution context.
- Do not write guards like `guard motionManager.isAccelerometerAvailable else { return }`. Throw a Nitro runtime error or reject the promise so JS can handle the failure.
- **Avoid silently swallowing not implemented functions.** If a feature is not available on a platform, throw an explicit not-available/not-implemented error.
- Do not use `NSError` or Objective-C types in general. Prefer Nitro's runtime error type, for example Swift's `RuntimeError("...")` from `NitroModules`.
- For impossible-to-reach states, you can use static assertions or `fatalError(...)` in Swift, but this should be super rare and not for stuff that can be reached via faulty JS user code.

```swift
import NitroModules

final class HybridMotion: HybridMotionSpec {
  var isAccelerometerAvailable: Bool {
    return motionManager.isAccelerometerAvailable
  }

  func startAccelerometerUpdates() throws {
    guard motionManager.isAccelerometerAvailable else {
      throw RuntimeError("Accelerometer is not available on this device.")
    }
    motionManager.startAccelerometerUpdates()
  }
}
```

## Native State and Object Shape

- **Nitro allows us to use native state (`HybridObject`).** This is especially useful to zero-copy bridge native data, for example large `UIImage`, zero-copy access via `ArrayBuffer`, or properties/methods on it.
- Use instance-based APIs and native state API design patterns. For haptics modules, instead of making everything a static method as you would have in a TurboModule, make the haptics engine an instance (`HybridObject`) and pre-warm the engine so that a call to trigger can be faster.
- For HybridObjects that require arguments to be constructed, use a factory pattern. For example, for a Nitro `File` object that needs to be created from a path string, create `FileFactory` with a method like `loadFileFromPath(path: string): Promise<File>`. Here, `Promise` is important because it is async. For non-async/fast methods use sync APIs.
- Use sync methods by default as that is super fast, but for anything that takes longer to execute, for example hardware calls or async APIs, use async methods (`Promise<...>`). In native code, use `Promise.parallel` for dispatch queue/thread based work or `Promise.async` for async/coroutine based work depending on the native threading requirements.
- Use descriptive method names. For helpers, prefer names like `timestampMs()` over vague names like `timestamp()`.

```swift
private static func timestampMs() -> Double {
  return Date().timeIntervalSince1970 * 1_000.0
}
```

## Memory, Views, and Buffers

- For types that hold onto native resources or have large memory allocations, implement `memorySize` (an overridable property in `HybridObject`) and try to estimate the size somewhat closely. For example, estimate a `UIImage` byte size by multiplying width, height, and bytes per pixel. This helps the JS VM / GC delete sooner and avoid memory stress.
- For Nitro Views, implement view recycling if applicable. Use `prepareForRecycle`, which is overridable from `HybridView`.
- For zero-copy data access, use `ArrayBuffer`; it can wrap many native buffer types.
- When receiving an `ArrayBuffer` from JS but you need to use it on a different thread, check if it is owning or not via `arrayBuffer.isOwner`. If not, copy it first: `let buffer = arrayBuffer.isOwner ? arrayBuffer : ArrayBuffer.copy(of: arrayBuffer)`.

## Platform Interop

- If you need Android context, use `NitroModules.applicationContext` and just throw if it is not available.
- If you need to mix C++ and platform languages (Swift/Kotlin), you can do so in Nitro. This allows re-using C++ code across iOS and Android.
- You can pass a HybridObject implemented in a platform language (for example Swift/Kotlin) to C++ and use it as normal in C++. This is useful for something like SQLite, where a `PlatformFilesystem` HybridObject has a `createFile(): string` method. JS can create `PlatformFilesystem` and pass it to a C++ SQLite HybridObject, and C++ can call `createFile()` directly even though it is implemented in Swift/Kotlin.
- Prefer Nitro Modules over JSI. Nitro is not only faster than TurboModules and often even faster than handwritten JSI, but also much safer: it avoids retaining JSI values across different threads, calling Promise resolve after runtime destruction, and similar crash-prone patterns.
- Resort to JSI only if absolutely needed via Raw JSI Methods (`prototype.registerRawHybridMethod` API from Nitro C++).

## TypeScript Layer

- In React Native environments, try to provide Hooks APIs on top of the imperative APIs. Use an initial getter (`sync` in `useMemo`/`useRef`/`useSyncExternalStore`, async via `useEffect` or similar) plus listener-based APIs with an unsubscribe function via `useEffect`.
- When creating TypeScript abstractions on top of native Nitro bindings, you can use TypeScript features like discriminating unions or nullables with default values more easily. Those things would have slight overhead and more type complexity in native Nitro code.
- Example: `takePhoto(options:)` might have a complex options struct. Making all of that optional causes more complex native code; simply making it non optional and providing default values via TypeScript (`??`) makes it simpler, easier to inline for the JS engine, and lower bridging cost.
- The goal should always be to provide simpler and more explicit native APIs. Don't overengineer this; if it is overcomplicating native code then it is not worth the minor performance gain.
- Use callbacks for asynchronous functions, and in super rare cases when a callback needs to run fully synchronously/blocking on the JS Thread, use `Sync<(...) => ...>`, for example for worklets interop. See `react-native-vision-camera` V5 `FrameOutput.nitro.ts` for an example on how to use `Sync` callbacks on Worklet Threads.

## Testing

- If available, use `react-native-harness` for end-to-end testing Nitro modules.
- Write Harness tests for real features to ensure that each individual feature, input/param combination, setting, property, combination, order of execution, and more works properly in a real React Native environment.
- Use Harness CI tests via GitHub Actions to continuously iterate until CI is green and to cover more API surface area / combinations.
- Test behaviour. There is no point in testing types like `toBeDefined`, `Array.isArray`, or `typeof`, because Nitro(gen) already enforces types at compile-time.

## When Unsure

Refer to Nitro's docs if anything else is unclear: <https://nitro.margelo.com/llms-full.txt>
