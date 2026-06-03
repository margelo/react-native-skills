---
name: swift
description: Design, implement, and review Swift APIs and Apple-platform code. Use when working on .swift files, Swift types, Foundation or AVFoundation APIs, DispatchQueue, async/await, Task, actors, MainActor, thread-affine state, or Swift-backed React Native Nitro Module implementations.
---

# Swift

Use this skill for Swift code that needs strong API boundaries, predictable threading, and idiomatic Apple-platform integration. When the code is part of a Nitro Module, pair this with `build-nitro-modules` for generated specs, Promise mapping, and HybridObject constraints.

## Workflow

1. Read the local Swift code, generated protocols, and surrounding API shape before editing.
2. Choose the public type model first: make invalid states unrepresentable where Swift can express them.
3. Choose one concurrency model for the feature before writing implementation code.
4. Keep synchronous properties and methods cheap, local, and nonblocking.
5. Make queue hops, hardware/session negotiation, I/O, and fallible async work explicit in the API.

## Type-Safe API Design

- Represent state variants with types, not nullable clusters. Use protocols plus conforming structs/classes when variants share a public contract, or use `enum` with associated values when the set is closed and value-like.
- Keep related fields nonoptional on the same variant. If `barcode` and `barcodeType` are meaningful only together, put both on `ScannedBarcode`; do not make both optional on a generic scanned-data struct.
- Use optionals only for real domain absence inside one state, not for expressing which state the object is in.
- Prefer compile-time flow over caller-side probing. If callers need repeated `if let` chains to discover valid field combinations, the API shape is probably wrong.

```swift
// Avoid: the valid combinations are implicit and easy to misuse.
struct ScannedData {
  let position: Point
  let text: String?
  let barcode: String?
  let barcodeType: BarcodeType?
  let face: Rect?
}

// Prefer: each state exposes the fields that are valid for that state.
protocol ScannedData {
  var position: Point { get }
}

struct ScannedText: ScannedData {
  let position: Point
  let text: String
}

struct ScannedBarcode: ScannedData {
  let position: Point
  let barcode: String
  let barcodeType: BarcodeType
}

struct ScannedFace: ScannedData {
  let position: Point
  let face: Rect
}
```

## Concurrency Model

- Choose Swift concurrency or DispatchQueue for a feature, not both as interleaved control flow.
- Use Swift `async`/`await`, `Task`, and actors only when the full operation can be represented cleanly in Swift concurrency without queue escape hatches.
- Use a private serial `DispatchQueue` when Apple APIs, delegates, callbacks, C++ bridges, JS runtimes, or Nitro thread boundaries already revolve around queues.
- Do not force `MainActor.assumeIsolated`, `Thread.isMainThread` branches, or queue `.sync` calls to make a Swift-concurrency design compile. That is a signal to choose a queue-based design or change the API boundary.
- Do not call `DispatchQueue.sync`, `DispatchQueue.main.sync`, or equivalent synchronous queue hops. Treat them as bugs, especially in property getters and setters.
- Use `DispatchQueue.async` or a Nitro `Promise` method for queue-bound work that callers must wait for.

## Properties and Threading

- Use properties only for cheap observed state that can be read or written immediately and safely.
- Do not hide a thread hop, hardware/session query, lock wait, permission check, allocation, or fallible native operation behind a property.
- If a value can only be observed from a specific queue, prefer an event/listener emitted from that queue.
- If a caller truly needs a one-shot read from another queue, expose an async getter method instead of a property.
- In Nitro Modules, represent queue-bound reads and writes as `Promise<T>` methods, usually with `Promise.parallel(queue)`.

```swift
// Avoid: synchronous queue hop hidden in a getter.
var status: SessionStatus {
  queue.sync { session.status }
}

// Prefer: make the async boundary explicit.
func getStatus() throws -> Promise<SessionStatus> {
  return Promise.parallel(queue) {
    return self.session.status
  }
}
```

## Swift Style

- Make classes `final` by default unless subclassing is part of the design.
- Prefer Swift-native types such as `String`, `Array`, `Dictionary`, structs, protocols, and Foundation value types. Avoid Objective-C bridge types unless an Apple API requires them.
- Use `guard` to validate input and state early. Throw specific errors for user-reachable failures.
- Treat a filename as a scope contract. `HybridDataScanner.swift` should implement `HybridDataScanner`; it should not also contain unrelated `CGPoint` conversions, `UIViewController` helpers, delegates, or framework adapters.
- Keep one primary type or cohesive extension family per file by default. Put reusable extensions in named files such as `Extensions/UIViewController+topPresentedViewController.swift` or `Extensions/CGPoint+Point.swift` with `internal` or `package` visibility where appropriate.
- Split delegates, framework adapters, converters, native protocols, and helper state into separate files instead of adding them below the main type.
- Use line count as a review signal: files below roughly 300 lines are usually fine; files above that need a clear reason tied to one cohesive responsibility. Size caused by unrelated extensions, conversions, or helper types is a design issue.
- Break meaningful conversions into named intermediate values instead of long inline expressions.

## Nitro Notes

- Use `Promise.parallel(queue)` for DispatchQueue-based work such as AVFoundation session operations.
- Use `Promise.async` only when wrapping Swift `async`/`await` or Task-native APIs end to end.
- Do not mix `Promise.async` with queue `.sync` calls or actor escape hatches.
- Generated HybridObject properties are synchronous entry points. Redesign them as methods or listeners if they need queue affinity.
