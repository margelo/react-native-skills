---
title: Writing HybridObject Specs and Exporting
impact: CRITICAL
tags: hybrid-object, spec, nitro.ts, typescript, NitroModules, export, interface
---

# Skill: Writing HybridObject Specs and Exporting

Covers Steps 4 and 10: deleting the default spec, writing a domain-specific `*.nitro.ts` spec, and exporting the HybridObject from runtime JS/TS.

## Quick Pattern

**Incorrect** — keeping the default stub:
```typescript
// src/specs/Example.nitro.ts  ← DELETE THIS
import { type HybridObject } from 'react-native-nitro-modules'
interface Example extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {}
```

**Correct** — domain-specific spec that exports only the interface:
```typescript
// src/specs/Math.nitro.ts
import type { HybridObject } from 'react-native-nitro-modules'

/**
 * Performs fast native math operations.
 */
export interface Math extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  /**
   * Adds two numbers synchronously.
   */
  add(a: number, b: number): number
  subtract(a: number, b: number): number
  multiply(a: number, b: number): Promise<number>
}
```

## When to Use

- After scaffolding with `nitrogen init`, before running `bunx nitrogen`
- After any API change to the module's interface
- When adding new methods or properties to an existing module

## Prerequisites

- Library scaffolded via `bunx nitrogen@latest init <name>`
- `packages/<name>/src/specs/` directory exists

## Step-by-Step

### 1. Delete the default spec

```bash
rm packages/react-native-math/src/specs/Example.nitro.ts
```

The scaffold creates `Example.nitro.ts` as a placeholder — always replace it with your domain-specific spec.

### 2. Create the spec file

Name it after the module's domain: `Math.nitro.ts`, `Camera.nitro.ts`, `Crypto.nitro.ts`.

**The `.nitro.ts` extension is required** — Nitrogen only parses files ending in `.nitro.ts`.

```bash
touch packages/react-native-math/src/specs/Math.nitro.ts
```

For non-trivial modules, split specs and shared types into focused files instead of one large catch-all file.

File boundary rules:
- Each primary HybridObject gets its own `.nitro.ts` file.
- Keep an inheritance family in one `.nitro.ts` file only when the file is named after the base HybridObject and child HybridObjects add few or no members.
- Put enums, literal unions, structs, option interfaces, event interfaces, callback types, and helper types in their own `.ts` files under focused folders such as `src/specs/common-types/`, `src/specs/barcodes/`, or another domain folder.
- Import helper types into `.nitro.ts` specs instead of defining them inline.
- Re-export public spec types and helper types from `src/index.ts`.
- Group multiple helper types in one file only when they form one tightly coupled logical construct and are rarely imported independently.

### 3. Write the interface

```typescript
import type { HybridObject } from 'react-native-nitro-modules'

export interface Math extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  // Synchronous methods
  add(a: number, b: number): number
  subtract(a: number, b: number): number

  // Async methods return Promise
  calculateFibonacci(n: number): Promise<number>

  // Properties (readable)
  readonly pi: number

  // Properties (readable + writable)
  precision: number

  // Optional parameters
  round(value: number, decimals?: number): number
}
```

### 4. Choose platform languages

In the `HybridObject<{ ... }>` generic:
- `ios: 'swift'` — iOS implemented in Swift
- `ios: 'c++'` — iOS implemented in C++ (cross-platform)
- `android: 'kotlin'` — Android implemented in Kotlin
- `android: 'c++'` — Android implemented in C++ (cross-platform)

For C++ only (both platforms): `HybridObject<{ ios: 'c++'; android: 'c++' }>`

> **Note:** Both the `.nitro.ts` spec and `nitro.json` autolinking use `"c++"`. In `nitro.json`, the C++ autolinking entry uses `"all": { "language": "c++", "implementationClassName": "HybridMath" }`.

### Mixed C++ and platform HybridObjects

A single Nitro library can mix C++ HybridObjects and platform-language HybridObjects. Use this when most code should be Swift/Kotlin but one subsystem needs shared C++, or when C++ needs platform services through a typed object.

```typescript
import type { HybridObject } from 'react-native-nitro-modules'

export interface Storage
  extends HybridObject<{ ios: 'c++'; android: 'c++' }> {}

export interface PlatformContext
  extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  getTemporaryDirectory(): string
  writeFile(content: ArrayBuffer, path: string): Promise<void>
}

export interface StorageFactory
  extends HybridObject<{ ios: 'c++'; android: 'c++' }> {
  createStorage(context: PlatformContext): Storage
}
```

In this pattern, `StorageFactory` is implemented in C++ and can call the generated C++ spec API for `PlatformContext`, even though `PlatformContext` is implemented in Swift/Kotlin. C++ sees public generated methods such as `getTemporaryDirectory()` and `writeFile(...)`; it cannot access private Swift/Kotlin implementation fields.

Use this for patterns such as:
- C++ frame processors using OpenCV while camera/session control stays in Swift/Kotlin.
- C++ storage, ML, image, audio, or compression engines that need platform paths, files, permissions, or OS handles from Swift/Kotlin objects.
- Cross-platform C++ pipelines where platform HybridObjects inject OS-specific services.

Do not assume the inverse direction is available. Before designing Swift/Kotlin code that directly consumes C++-implemented HybridObjects, verify current Nitrogen support.

### 5. Export the HybridObject (Step 10)

After implementing native code, create and export the HybridObject from `src/index.ts`:

```typescript
// src/index.ts
import { NitroModules } from 'react-native-nitro-modules'
import type { Math } from './specs/Math.nitro'

export const math = NitroModules.createHybridObject<Math>('Math')
export type { Math } from './specs/Math.nitro'
```

**Rules:**
- Always export the runtime HybridObject value from normal TypeScript, such as `export const math = ...`. This is the value app code uses at runtime.
- Define the spec type in the `.nitro.ts` file with `export interface Math ...`.
- If consumers should import the spec type from the package root, re-export it from `index.ts` with `export type { Math } from './specs/Math.nitro'`. This is only an additional type export; it does not replace the runtime value export.
- The string `'Math'` in `createHybridObject<Math>('Math')` must exactly match the key in `nitro.json`'s `autolinking` block
- Prefer naming native classes with the `Hybrid` prefix: `HybridMath`
- Keep both the interface name and the autolinking key the same (e.g. `Math` = `'Math'`)
- For larger libraries, create one autolinked factory/root object and return other stateful HybridObjects from factory methods instead of autolinking every object.
- Do not put a HybridObject and all related enums, options, results, events, and helper structs in one `.nitro.ts` file. Split them into focused files and import them.
- If creating a returned object requires setup, I/O, permission checks, or validation that can fail, make the factory method async and return a ready object. Do not expose `prepare()`/`initialize()` methods that callers must remember before normal use.
- Add JSDoc to every exported spec interface, type alias, string-literal union, callback, options struct, event struct, HybridObject, and public property. Type-level comments must describe domain meaning and link to a real related type or member with `{@linkcode ...}` or `@see`, for example `Represents the format of a {@linkcode Barcode}.` and `@see {@linkcode Barcode.format}`. Do not invent link targets.
- Listener methods must return a flat subscription object with `remove(): void`. Do not make the subscription a HybridObject unless it exposes native state beyond cleanup. Do not expose numeric listener IDs or `removeListener(listenerId)` methods.

## Code Examples

### Module with properties and listener callbacks

```typescript
import type { HybridObject } from 'react-native-nitro-modules'

export interface ListenerSubscription {
  remove(): void
}

export interface Camera extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  // Properties
  readonly isRecording: boolean
  zoom: number

  // Listener callbacks
  addOnFrameCapturedListener(
    listener: (frame: ArrayBuffer) => void,
  ): ListenerSubscription

  // Async methods
  startRecording(): Promise<void>
  stopRecording(): Promise<string>  // returns file path

  // Sync methods
  setFlashMode(mode: 'on' | 'off' | 'auto'): void
}
```

Then create it in `src/index.ts`:

```typescript
import { NitroModules } from 'react-native-nitro-modules'
import type { Camera } from './specs/Camera.nitro'

export const camera = NitroModules.createHybridObject<Camera>('Camera')
```

### Factory-root pattern for stateful objects

Autolink the public root/factory, then create stateful resource objects through methods:

```typescript
import type { HybridObject } from 'react-native-nitro-modules'

export interface VideoOutputOptions {
  targetBitRate?: number
}

export interface Recorder extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  readonly isRecording: boolean
  startRecording(
    onFinished: (filePath: string) => void,
    onError: (error: Error) => void,
  ): Promise<void>
  stopRecording(): Promise<void>
}

export interface VideoOutput extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  createRecorder(filePath?: string): Promise<Recorder>
}

export interface MediaFactory extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  createVideoOutput(options: VideoOutputOptions): VideoOutput
}
```

In this pattern, `MediaFactory` is autolinked because JS creates it directly. Do not add `nitro.json` autolinking entries for `VideoOutput` or `Recorder` unless JS creates them directly.

### Multiple native implementations for one spec

Use one JS-facing HybridObject spec when multiple native backends should behave the same from TypeScript. Implement the generated spec in multiple native classes, choose the concrete class in a factory, and return the shared spec type.

Example: `CameraVideoOutput` can be backed by `HybridMovieFileVideoOutput` using a platform movie-file output or by `HybridAssetWriterVideoOutput` using a video data output plus asset writer. Both native classes implement the generated `HybridCameraVideoOutputSpec`, and JS only sees `CameraVideoOutput`.

Rules:
- Keep backend class names native-only unless callers must choose a backend explicitly.
- Return the shared spec type from factory methods, such as `createVideoOutput(...): Promise<CameraVideoOutput>`.
- Add one `nitro.json` autolinking entry only for HybridObjects JS creates directly. Do not autolink every concrete backend returned by a factory.
- If native session code needs backend-specific handles, pair the JS-facing spec with a native protocol/interface and make each concrete implementation conform to both.

If constructing `VideoOutput` needs native setup or can fail, make the factory async instead: `createVideoOutput(options: VideoOutputOptions): Promise<VideoOutput>`. The returned object should already be usable.

### HybridObject inheritance for result families

Use HybridObject inheritance when a feature returns heterogeneous native objects that share identity, lifecycle, coordinates, timestamps, raw native handles, or common methods.

```typescript
import type { HybridObject } from 'react-native-nitro-modules'
import type { Bounds } from '../common-types/Bounds'
import type { BarcodeFormat } from '../barcodes/BarcodeFormat'

export type ScannedItemType = 'barcode' | 'qr-code' | 'face'

export interface ScannedItem
  extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  readonly id: string
  readonly type: ScannedItemType
  readonly bounds: Bounds
}

export interface ScannedBarcode extends ScannedItem {
  readonly format: BarcodeFormat
  readonly rawValue: string
}

export interface ScannedQRCode extends ScannedBarcode {
  readonly url?: string
}

export interface ScannedFace extends ScannedItem {
  readonly trackingId?: number
}
```

Rules:
- Put common properties and methods on the base HybridObject.
- Put subtype-only properties and methods on child HybridObjects.
- Return the base type for heterogeneous collections, such as `ScannedItem[]`.
- Add a discriminator property such as `type` so JS can narrow to the child type.
- Use this instead of one flat struct with every subtype field optional.
- Native code can accept the generated base spec when it needs common behavior, or the generated child spec when it needs subtype behavior.
- Keep thin inheritance families in the base `.nitro.ts` file. Move child HybridObjects to their own `.nitro.ts` file when they gain substantial API surface.

Export the factory under a product/domain object name rather than the spec type name:

```typescript
// src/Media.ts
import { NitroModules } from 'react-native-nitro-modules'
import type { MediaFactory } from './specs/MediaFactory.nitro'

export const Media =
  NitroModules.createHybridObject<MediaFactory>('MediaFactory')
```

The autolinking key remains `MediaFactory` because it matches the generated spec and `nitro.json`. The JS export should not mechanically become `mediaFactory`, `MediaFactory`, or `HybridMediaFactory`. Use a product/domain noun such as `VisionCamera`, `Images`, or `Media`.

### Native protocol extension points

Use this pattern when third-party native code should extend the library while still passing objects through JS/TS as typed HybridObjects.

Define a small JS-facing base spec that represents what can be passed through JS APIs:

```typescript
// src/specs/outputs/MediaOutput.nitro.ts
import type { HybridObject } from 'react-native-nitro-modules'

/**
 * Base interface for outputs that can be connected to a MediaSession.
 *
 * Native implementations can conform to the platform-native
 * `NativeMediaOutput` protocol/interface to expose their real native handle.
 */
export interface MediaOutput extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  readonly mediaType: 'video' | 'audio'
}
```

Then define a public native protocol/interface that exposes native-only handles and behavior:

```swift
// ios/Public/NativeMediaOutput.swift
import AVFoundation

public protocol NativeMediaOutput: AnyObject {
  associatedtype Output: AVCaptureOutput
  var output: Output { get }
  func configure(config: MediaOutputConfiguration)
}
```

```kotlin
// android/src/main/java/com/example/media/public/NativeMediaOutput.kt
import androidx.camera.core.UseCase

interface NativeMediaOutput {
  fun createUseCase(config: MediaOutputConfig): UseCase
}
```

First-party and third-party outputs conform to both the generated Nitro spec and the native protocol/interface:

```swift
final class HybridPhotoOutput: HybridMediaOutputSpec, NativeMediaOutput {
  let mediaType: MediaType = .video
  let output = AVCapturePhotoOutput()

  func configure(config: MediaOutputConfiguration) {
    // Apply orientation, mirroring, resolution, or platform-specific settings.
  }
}
```

Native session/controller code should accept the generated base spec from JS, then unwrap it through the native protocol/interface:

```swift
func addOutput(_ output: any HybridMediaOutputSpec) throws {
  guard let nativeOutput = output as? any NativeMediaOutput else {
    throw RuntimeError.error(withMessage: "Output is not a NativeMediaOutput!")
  }
  session.addOutput(nativeOutput.output)
}
```

This is the same shape VisionCamera uses for `CameraOutput` and `NativeCameraOutput`: JS sees a portable `CameraOutput`, while native code can unwrap an `AVCaptureOutput` on iOS or a CameraX `UseCase` on Android. Keep the native protocol/interface in public source folders, document it, and publish it with the npm package so third-party native modules can conform to it.

### TypeScript → Native type mapping

| TypeScript | C++ | Kotlin | Swift |
|-----------|-----|--------|-------|
| `number` | `double` | `Double` | `Double` |
| `string` | `std::string` | `String` | `String` |
| `boolean` | `bool` | `Boolean` | `Bool` |
| `bigint` | `int64_t` / `uint64_t` | `Long` / `ULong` | `Int64` / `UInt64` |
| `T[]` | `std::vector<T>` | `Array<T>` | `[T]` |
| `Promise<T>` | `std::shared_ptr<Promise<T>>` | `Promise<T>` | `Promise<T>` |
| `T \| undefined` | `std::optional<T>` | `T?` | `T?` |
| `(x: T) => void` | `std::function<void(T)>` | `(T) -> Unit` | `(T) -> Void` |
| `ArrayBuffer` | `std::shared_ptr<ArrayBuffer>` | `ArrayBuffer` | `ArrayBuffer` |

## Common Pitfalls

- **Wrong file extension** — Must be `.nitro.ts`, not `.ts` or `.d.ts`
- **Mismatch between interface name and autolinking key** — `createHybridObject<Math>('Math')` string must match `nitro.json`
- **Forgetting platform languages** — `HybridObject<{}>` without specifying ios/android will fail
- **Creating runtime objects inside specs** — Keep `.nitro.ts` files focused on exported types; create the runtime object in `src/index.ts`
- **Missing export** — The hybrid object won't be usable from JS without the `createHybridObject` call and export

## Related Skills

- [spec-nitro-json.md](spec-nitro-json.md) — Configure autolinking to match the interface name
- [native-nitrogen-codegen.md](native-nitrogen-codegen.md) — Run codegen after writing the spec
