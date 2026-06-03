---
title: Implementing HybridObjects in Swift
impact: HIGH
tags: swift, ios, hybrid-object, native, implementation, NitroModules, protocol, class
---

# Skill: Implementing HybridObjects in Swift

Covers Steps 8–9 (Swift path): creating the Swift implementation class that implements the Nitrogen-generated iOS spec protocol/base typealias.

## Quick Pattern

**Incorrect** — subclassing NSObject directly:
```swift
import Foundation
class HybridMath: NSObject {
  func add(a: Double, b: Double) -> Double { a + b }
}
```

**Correct** — implementing the generated spec:
```swift
import Foundation
import NitroModules

final class HybridMath: HybridMathSpec {
  func add(a: Double, b: Double) throws -> Double { a + b }
}
```

## When to Use

- Implementing the iOS side of a Nitro module in Swift
- When the spec uses `{ ios: 'swift' }`
- After Nitrogen has generated `HybridMathSpec.swift`

## Prerequisites

- Nitrogen has generated `HybridMathSpec.swift`, usually in `nitrogen/generated/ios/swift/`
- `nitro.json` has an iOS autolinking entry with `"language": "swift"` and `"implementationClassName": "HybridMath"`
- `react-native-nitro-modules` is a pod dependency

## Step-by-Step

### 1. Locate the generated spec

```
nitrogen/generated/ios/swift/HybridMathSpec.swift   ← generated protocol/base spec
```

### 2. Create the implementation file

```bash
touch ios/HybridMath.swift
```

### 3. Write the implementation class

```swift
import Foundation
import NitroModules

final class HybridMath: HybridMathSpec {
  private static let queue = DispatchQueue(label: "com.margelo.math")

  // Synchronous methods — most generated methods have `throws`
  func add(a: Double, b: Double) throws -> Double {
    return a + b
  }

  func subtract(a: Double, b: Double) throws -> Double {
    return a - b
  }

  // Async method returning Promise — also has `throws`
  func calculateFibonacci(n: Double) throws -> Promise<Double> {
    return Promise.parallel(Self.queue) {
      if n <= 1 { return n }

      var previous = 0.0
      var current = 1.0
      for _ in 2...Int(n) {
        let next = previous + current
        previous = current
        current = next
      }
      return current
    }
  }

  // Readonly property
  var pi: Double { Double.pi }

  // Read-write property
  var precision: Double = 6.0
}
```

### 4. Add to the podspec

In the package podspec, ensure the implementation file and generated files are included. Current Nitro templates usually do this through `add_nitrogen_files(s)`:

```ruby
load 'nitrogen/generated/ios/NitroMath+autolinking.rb'
add_nitrogen_files(s)
```

If the podspec manually lists source files, include `ios/**/*.{h,m,mm,swift}`, `nitrogen/generated/ios/**/*.{h,hpp,cpp,mm,swift}`, and any shared `cpp/**/*.{hpp,cpp}` files.

### 5. Verify using canonical Swift reference

For any type uncertainty, consult the canonical Swift test implementation:
[HybridTestObjectSwift.swift](https://github.com/mrousavy/nitro/blob/main/packages/react-native-nitro-test/ios/HybridTestObjectSwift.swift)

## Code Examples

### Type reference table

| TypeScript | Swift Type | Notes |
|-----------|-----------|-------|
| `number` | `Double` | Always `Double` |
| `string` | `String` | |
| `boolean` | `Bool` | |
| `bigint` (signed) | `Int64` | |
| `bigint` (unsigned) | `UInt64` | |
| `T[]` | `[T]` | e.g. `[Double]`, `[String]`, `[Person]` |
| `Promise<T>` | `Promise<T>` | `Promise.parallel(queue) { }`, `Promise.async { }`, `Promise.resolved(withResult:)`, or manual `Promise()` |
| `Promise<void>` | `Promise<Void>` | Swift `Void`, not `Unit` |
| `T \| undefined` | `T?` | Swift optional |
| `T \| U` | `Variant_T_U` | Generated type, e.g. `Variant_String_Double` |
| `(x: T) -> void` | `@escaping (T) -> Void` | Must be `@escaping` for stored callbacks |
| `() -> T` | `@escaping () -> T` | |
| `ArrayBuffer` | `ArrayBuffer` | From NitroModules |
| `AnyMap` | `AnyMap` | From NitroModules |
| `Record<string, T>` | `[String: T]` | Swift dictionary literal syntax |
| `HybridObject` | `any HybridSpec` | Protocol existential, e.g. `any HybridMathSpec` |
| `null` / `NullType` | `NullType` | `.null` value |
| `Date` | `Date` | Foundation `Date` |

### Async with Promise

Choose one concurrency model for the feature before implementing it.

- Use `Promise.parallel(queue)` for DispatchQueue-based work. This fits most AVFoundation and session-style APIs because you can keep all native state mutations on one queue.
- Use `Promise.async` only when the operation is naturally Swift `async`/`await` or Task-based from end to end.
- Do not mix Swift concurrency with `DispatchQueue.sync`, `DispatchQueue.main.sync`, `MainActor.assumeIsolated`, or `Thread.isMainThread` workarounds. If those seem necessary, use a queue-based design or change the public API boundary.
- Never call `DispatchQueue.sync` or `DispatchQueue.main.sync` in Nitro implementation code. This is especially dangerous in generated property getters and setters because those are synchronous JS entry points.

```swift
private static let cameraQueue = DispatchQueue(label: "com.margelo.camera.session")

func calculateFibonacciAsync(value: Double) throws -> Promise<Int64> {
  return Promise.parallel(Self.cameraQueue) {
    return try self.calculateFibonacciSync(value: value)
  }
}
```

Use `Promise.async` when you are intentionally wrapping Swift `async`/`await` or an API that naturally runs through `Task`.

```swift
func loadRemoteImage(url: URL) throws -> Promise<Data> {
  return Promise.async {
    let request = URLRequest(url: url)
    let result = try await URLSession.shared.data(for: request)
    return result.0
  }
}

// Promise<Void> — void async (Swift uses Void not Unit)
func wait(seconds: Double) throws -> Promise<Void> {
  return Promise.async {
    let secondsUInt64 = UInt64(seconds)
    let nanoseconds = secondsUInt64 * 1_000_000_000
    try await Task.sleep(nanoseconds: nanoseconds)
  }
}

// Promise.resolved — instant resolution with a value
func promiseReturnsInstantly() throws -> Promise<Double> {
  return Promise.resolved(withResult: 55.0)
}

// Promise.resolved() — instant void resolution
func promiseThatResolvesVoidInstantly() throws -> Promise<Void> {
  return Promise.resolved()
}

// Promise<T?> — resolves to undefined/nil
func promiseThatResolvesToUndefined() throws -> Promise<Double?> {
  return Promise.resolved(withResult: nil)
}
```

### Threading and state

HybridObject methods and property access can be called synchronously from JS, including from multiple JS runtimes such as worklets. Do not model HybridObject implementations as Swift `actor`s by default; actors make sync generated methods awkward and can hide where serialization happens.

Prefer one of these patterns:
- Keep cheap readonly state synchronous.
- Put mutable native/session state behind a private serial `DispatchQueue`, and run mutating or fallible operations through `Promise.parallel(queue)`.
- Make operations async when they must serialize, wait for hardware/session state, or cross queues.
- Emit listener events from the queue/thread that owns the state when callers need ongoing observations.
- Add `NSLock` or another lock only after shared mutable state is proven to be accessed concurrently and a queue-owned design is not a better fit.

### Using callbacks

```swift
func compute(input: Double, onResult: @escaping (Double) -> Void) {
  let result = input * 2.0
  onResult(result)
}
```

### Handling optional parameters

```swift
var optionalValue: Double? = nil

func round(value: Double, decimals: Double?) -> Double {
  let places = decimals ?? 0
  let multiplier = pow(10.0, places)
  return Foundation.round(value * multiplier) / multiplier
}
```

### Throwing errors

Use `guard` for state/input validation. Throw `RuntimeError` for user-reachable failures. Do not expose `NSError` paths unless a generated API or Apple callback forces it, and convert those errors before they cross into JS.

```swift
func divide(a: Double, b: Double) throws -> Double {
  guard b != 0 else {
    throw RuntimeError("Division by zero!")
  }
  return a / b
}
```

### Validating configuration

Validate invalid values and required behavior early, but do not reject optional cross-platform preferences only because iOS ignores them. If the operation still does its core job, ignore or degrade the preference and report support through the public capabilities or resolved state.

Examples:
- Throw when a requested flash mode cannot work because the device has no flash.
- Do not throw only because a quality, guidance UI, high-frame-rate, auto-zoom, or region preference is unsupported, unless the API documents that field as a hard requirement.

### Properties and thread affinity

```swift
private var _zoom: Double = 1.0

var zoom: Double {
  get { _zoom }
  set {
    _zoom = newValue
  }
}
```

Writable properties are synchronous JS calls. Keep getters and setters cheap, local, and unlikely to fail. If applying a value requires queue hops, AVFoundation negotiation, permissions, allocation, or can fail, expose a method that returns `Promise<Void>` instead.

```swift
// Avoid: synchronous queue hop hidden in a getter.
var status: SessionStatus {
  queue.sync { session.status }
}

// Prefer: make the queue boundary explicit.
func getStatus() throws -> Promise<SessionStatus> {
  return Promise.parallel(Self.cameraQueue) {
    return self.session.status
  }
}

func setZoom(_ zoom: Double) throws -> Promise<Void> {
  return Promise.parallel(Self.cameraQueue) {
    try self.applyZoomToCamera(zoom)
  }
}
```

If state can only be observed on a specific queue, prefer a listener or event API emitted from that queue instead of a getter.

### Swift style and organization

- Make HybridObject implementation classes `final` unless inheritance is genuinely required.
- Use Swift types such as `String`, `[String: T]`, arrays, structs, and typed Foundation values. Avoid Objective-C bridge types such as `NSString`, `NSDictionary`, `NSArray`, and `NSObject` inheritance unless an Apple API requires them.
- Treat `ios/HybridDataScanner.swift` as the implementation file for `HybridDataScanner`, not as a dumping ground for unrelated Swift extensions, UI helpers, geometry conversions, delegates, or native protocols.
- Put reusable conversions, Apple framework helpers, and small protocol conveniences in Swift extensions under `ios/Extensions/`, such as `ios/Extensions/UIViewController+topPresentedViewController.swift`, `ios/Extensions/CGPoint+Point.swift`, or `ios/Extensions/AVFoundation/AVCaptureDevice+withLock.swift`.
- Move delegates, framework adapters, converters, native protocols, and helper state into separate named files. Use `internal` or `package` visibility when the helper should not be public API.
- Use line count as a review signal: under roughly 300 lines is usually acceptable, while files above that need a concrete reason tied to one cohesive responsibility. A large file caused by extension methods or helper variables belongs in multiple files.
- Put one-element conversions on the source type. A Vision conversion should live in a file such as `ios/Extensions/RecognizedDataType+DataScannerRecognizedDataType.swift`; it can return `[DataScannerViewController.RecognizedDataType]` or `Set<DataScannerViewController.RecognizedDataType>` when one source value expands to several native values.
- Compose collections where they are used, for example `Set(try dataTypes.flatMap { try $0.toVisionRecognizedDataTypes() })`. Keep an aggregate helper only when it owns real collection semantics such as deduplication, validation across elements, nonempty checks, batching, caching, or error aggregation, and place it in a focused conversion/configuration file rather than `HybridDataScanner.swift`.
- Break complex expressions into named intermediate values. Avoid inline chains that allocate, convert units, and call another API in one expression.
- Pass named constants or variables into API calls instead of building values inline when the expression has meaningful steps.

```swift
let radians = angleDegrees * .pi / 180.0
let rotatedPoint = point.rotated(by: radians)
let projectedPoint = projection.project(rotatedPoint)
renderer.render(point: projectedPoint)
```

## Common Pitfalls

- **Forgetting `import NitroModules`** — The spec protocol won't be found without this import
- **Subclassing `NSObject` instead of the spec** — `class HybridMath: NSObject` won't satisfy the generated protocol
- **Method signature mismatch** — Every parameter name and type must exactly match the generated spec
- **Forgetting `throws` keyword** — Most generated methods have `throws`; check the generated spec to confirm which ones do
- **`Promise<void>` vs `Promise<Void>`** — Swift uses `Void` (not Kotlin's `Unit`). Always `Promise<Void>`
- **Using `Promise.async` for DispatchQueue APIs** — Prefer `Promise.parallel(queue)` for AVFoundation/session work. Use `Promise.async` for Swift `async`/`await` or Task-based APIs.
- **Using `DispatchQueue.sync` or `DispatchQueue.main.sync`** — Treat synchronous queue hops as a design bug. Make the Nitro API async or event-based instead.
- **Mixing actors and queues with escape hatches** — Do not use `MainActor.assumeIsolated`, `Thread.isMainThread`, or queue sync calls to force an async/await design onto queue-owned APIs.
- **Callbacks without `@escaping`** — Stored/async callbacks must be `@escaping`; the generated spec will tell you
- **`Dictionary<String,T>` vs `[String:T]`** — Both work; `[String:T]` is the idiomatic Swift syntax
- **`any HybridSpec` not `HybridSpec`** — In modern Swift, protocol types need the `any` keyword
- **Not including the file in podspec** — Swift files must be in the `source_files` glob in `.podspec`
- **Using the `override` keyword** — Swift implementations conform to the generated spec shape; methods and properties declared by the spec must NOT use `override` (unlike the Kotlin counterpart, which does). `override` only applies when overriding a superclass member.
- **Defaulting HybridObjects to `actor`** — JS-facing methods and properties are synchronous entry points. Prefer queue-owned state and async methods where serialization is needed.
- **Leaking Objective-C types** — Avoid `NSDictionary`, `NSString`, `NSArray`, and `NSError` in Nitro implementation APIs unless required by an Apple API boundary.
- **Letting one HybridObject file absorb every helper** — Split extensions, delegates, converters, and protocols into named files. The filename should still describe the file after the implementation is done.
- **Putting trivial maps behind collection extensions** — Prefer an element conversion plus `map`/`flatMap` at the call site. A collection helper is justified only when the collection itself adds behavior such as deduplication or validation.

## Related Skills

- [swift](../../swift/SKILL.md) — General Swift API, concurrency, and threading guidance
- [native-nitrogen-codegen.md](native-nitrogen-codegen.md) — Must generate specs before implementing
- [spec-nitro-json.md](spec-nitro-json.md) — Configure `"swift"` in autolinking
- [native-implement-kotlin.md](native-implement-kotlin.md) — Android Kotlin counterpart
- [native-implement-cpp.md](native-implement-cpp.md) — C++ cross-platform alternative
