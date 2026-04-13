---
title: Implementing HybridObjects in Swift
impact: HIGH
tags: swift, ios, hybrid-object, native, implementation, NitroModules, protocol, class
---

# Skill: Implementing HybridObjects in Swift

Covers Steps 8–9 (Swift path): creating the Swift implementation class that extends the Nitrogen-generated iOS spec.

## Quick Pattern

**Incorrect** — subclassing NSObject directly:
```swift
import Foundation
class HybridMath: NSObject {
  func add(a: Double, b: Double) -> Double { a + b }
}
```

**Correct** — extending the generated spec:
```swift
import NitroModules

class HybridMath: HybridMathSpec {
  override func add(a: Double, b: Double) throws -> Double { a + b }
}
```

## When to Use

- Implementing the iOS side of a Nitro module in Swift
- When the spec uses `{ ios: 'swift' }`
- After Nitrogen has generated `HybridMathSpec.swift`

## Prerequisites

- Nitrogen has generated `HybridMathSpec.swift` in `nitrogen/generated/ios/`
- `nitro.json` has `"swift": "HybridMath"` in the autolinking block
- `react-native-nitro-modules` is a pod dependency

## Step-by-Step

### 1. Locate the generated spec

```
nitrogen/generated/ios/HybridMathSpec.swift   ← abstract class, DO NOT EDIT
```

### 2. Create the implementation file

```bash
touch ios/HybridMath.swift
```

### 3. Write the implementation class

```swift
import NitroModules

class HybridMath: HybridMathSpec {

  // Synchronous methods — most generated methods have `throws`
  override func add(a: Double, b: Double) throws -> Double {
    return a + b
  }

  override func subtract(a: Double, b: Double) throws -> Double {
    return a - b
  }

  // Async method returning Promise — also has `throws`
  override func calculateFibonacci(n: Double) throws -> Promise<Double> {
    return Promise.async {
      if n <= 1 { return n }
      var a = 0.0, b = 1.0
      for _ in 2...Int(n) {
        let temp = a + b; a = b; b = temp
      }
      return b
    }
  }

  // Readonly property
  override var pi: Double { Double.pi }

  // Read-write property
  override var precision: Double = 6.0
}
```

### 4. Add to the podspec

In `ios/ReactNativeMath.podspec`, ensure the `HybridMath.swift` file is included:

```ruby
s.source_files = "ios/**/*.{h,m,mm,swift}", "nitrogen/generated/ios/**/*.{swift}", "cpp/**/*.{hpp,cpp}"
```

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
| `Promise<T>` | `Promise<T>` | `Promise.async { }`, `.async { }`, or `Promise.resolved(withResult:)` |
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

```swift
// Promise.async — async work with await support
override func calculateFibonacciAsync(value: Double) throws -> Promise<Int64> {
  return Promise.async { return try self.calculateFibonacciSync(value: value) }
}

// Promise<Void> — void async (Swift uses Void not Unit)
override func wait(seconds: Double) throws -> Promise<Void> {
  return Promise.async { try await Task.sleep(nanoseconds: UInt64(seconds) * 1_000_000_000) }
}

// Promise.resolved — instant resolution with a value
override func promiseReturnsInstantly() throws -> Promise<Double> {
  return Promise.resolved(withResult: 55.0)
}

// Promise.resolved() — instant void resolution
override func promiseThatResolvesVoidInstantly() throws -> Promise<Void> {
  return Promise.resolved()
}

// Promise<T?> — resolves to undefined/nil
override func promiseThatResolvesToUndefined() throws -> Promise<Double?> {
  return Promise.resolved(withResult: nil)
}
```

### Using callbacks

```swift
override func compute(input: Double, onResult: @escaping (Double) -> Void) {
  let result = input * 2.0
  onResult(result)
}
```

### Handling optional parameters

```swift
override var optionalValue: Double? = nil

override func round(value: Double, decimals: Double?) -> Double {
  let places = decimals ?? 0
  let multiplier = pow(10.0, places)
  return Foundation.round(value * multiplier) / multiplier
}
```

### Throwing errors

```swift
override func divide(a: Double, b: Double) throws -> Double {
  guard b != 0 else {
    throw NSError(domain: "Math", code: 1, userInfo: [NSLocalizedDescriptionKey: "Division by zero!"])
  }
  return a / b
}
```

### Properties with side effects

```swift
private var _zoom: Double = 1.0

override var zoom: Double {
  get { _zoom }
  set {
    _zoom = newValue
    applyZoomToCamera(newValue)
  }
}
```

## Common Pitfalls

- **Forgetting `import NitroModules`** — The spec class won't be found without this import
- **Extending `NSObject` instead of the spec** — `class HybridMath: NSObject` won't satisfy the generated protocol
- **Method signature mismatch** — Every parameter name and type must exactly match the generated spec
- **Forgetting `throws` keyword** — Most generated methods have `throws`; check the generated spec to confirm which ones do
- **`Promise<void>` vs `Promise<Void>`** — Swift uses `Void` (not Kotlin's `Unit`). Always `Promise<Void>`
- **Callbacks without `@escaping`** — Stored/async callbacks must be `@escaping`; the generated spec will tell you
- **`Dictionary<String,T>` vs `[String:T]`** — Both work; `[String:T]` is the idiomatic Swift syntax
- **`any HybridSpec` not `HybridSpec`** — In modern Swift, protocol types need the `any` keyword
- **Not including the file in podspec** — Swift files must be in the `source_files` glob in `.podspec`
- **Missing `override` keyword** — All overridden methods need `override`

## Related Skills

- [native-nitrogen-codegen.md](native-nitrogen-codegen.md) — Must generate specs before implementing
- [spec-nitro-json.md](spec-nitro-json.md) — Configure `"swift"` in autolinking
- [native-implement-kotlin.md](native-implement-kotlin.md) — Android Kotlin counterpart
- [native-implement-cpp.md](native-implement-cpp.md) — C++ cross-platform alternative
