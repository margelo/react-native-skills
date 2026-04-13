---
title: Implementing HybridObjects in Kotlin
impact: HIGH
tags: kotlin, android, hybrid-object, native, implementation, annotations, Keep, DoNotStrip, Promise
---

# Skill: Implementing HybridObjects in Kotlin

Covers Steps 8–9 (Kotlin path): creating the Kotlin implementation class that extends the Nitrogen-generated Android spec.

## Quick Pattern

**Incorrect** — missing required annotations:
```kotlin
class HybridMath : HybridMathSpec() {
  override fun add(a: Double, b: Double) = a + b
}
```

**Correct** — with required annotations:
```kotlin
@Keep
@DoNotStrip
class HybridMath : HybridMathSpec() {
  override fun add(a: Double, b: Double): Double = a + b
}
```

## When to Use

- Implementing the Android side of a Nitro module in Kotlin
- When the spec uses `{ android: 'kotlin' }`
- After Nitrogen has generated `HybridMathSpec.kt`

## Prerequisites

- Nitrogen has generated `HybridMathSpec.kt` in `nitrogen/generated/android/com/margelo/nitro/<namespace>/`
- `nitro.json` has `"kotlin": "HybridMath"` in the autolinking block

## Step-by-Step

### 1. Locate the generated spec

```
nitrogen/generated/android/com/margelo/nitro/math/HybridMathSpec.kt
```

This is the abstract class your implementation must extend. Do not edit it.

### 2. Create the implementation file

```bash
touch android/src/main/java/com/margelo/nitro/math/HybridMath.kt
```

The package must match `androidNamespace` from `nitro.json`.

### 3. Write the implementation class

```kotlin
package com.margelo.nitro.math

import androidx.annotation.Keep
import com.facebook.proguard.annotations.DoNotStrip

@Keep
@DoNotStrip
class HybridMath : HybridMathSpec() {

  // Synchronous method
  override fun add(a: Double, b: Double): Double = a + b

  override fun subtract(a: Double, b: Double): Double = a - b

  // Async method using Promise
  override fun calculateFibonacci(n: Double): Promise<Double> {
    return Promise.async {
      var a = 0.0; var b = 1.0
      for (i in 2..n.toInt()) {
        val temp = a + b; a = b; b = temp
      }
      b
    }
  }

  // Readonly property
  override val pi: Double = Math.PI

  // Read-write property
  override var precision: Double = 6.0
}
```

### 4. Add annotations — this is non-negotiable

- `@Keep` — Prevents ProGuard/R8 from removing the class
- `@DoNotStrip` — Prevents Meta's code stripper from removing it

Without both annotations, the class will be silently removed in release builds and cause `NullPointerException` at runtime.

### 5. Verify using canonical Kotlin reference

For any type uncertainty, consult the canonical Kotlin test implementation:
[HybridTestObjectKotlin.kt](https://github.com/mrousavy/nitro/blob/main/packages/react-native-nitro-test/android/src/main/java/com/margelo/nitro/test/HybridTestObjectKotlin.kt)

## Code Examples

### Type reference table

| TypeScript | Kotlin Type | Notes |
|-----------|-------------|-------|
| `number` | `Double` | Always `Double` |
| `string` | `String` | |
| `boolean` | `Boolean` | |
| `bigint` (signed) | `Long` | |
| `bigint` (unsigned) | `ULong` | Critical: `ULong`, not `Long` |
| `number[]` | `DoubleArray` | Primitive array — NOT `Array<Double>` |
| `T[]` (non-number) | `Array<T>` | e.g. `Array<String>`, `Array<Person>` |
| `Promise<T>` | `Promise<T>` | Use `Promise.async { }` or `Promise.parallel { }` |
| `Promise<void>` | `Promise<Unit>` | Kotlin `Unit`, not `Void` |
| `T \| undefined` | `T?` | Kotlin nullable |
| `(x: T) => void` | `(T) -> Unit` | Lambda type, no `@escaping` needed |
| `() => T` | `() -> T` | |
| `ArrayBuffer` | `ArrayBuffer` | From nitro-modules core |
| `AnyMap` | `AnyMap` | From nitro-modules core |
| `Record<string, T>` | `Map<String, T>` | |
| `HybridObject` | `HybridSpec` | Kotlin class, no `shared_ptr` |
| `null` / `NullType` | `NullType` | `NullType.NULL` constant |
| `Date` | `java.time.Instant` | |

### Async with Promise

```kotlin
// Promise.async — for IO-bound or suspending work (uses coroutines)
override fun wait(seconds: Double): Promise<Unit> {
  return Promise.async { delay(seconds.toLong() * 1000) }
}

// Promise.parallel — for CPU-bound synchronous work (runs on thread pool)
override fun calculateFibonacciAsync(value: Double): Promise<Long> {
  return Promise.parallel { calculateFibonacciSync(value) }
}

// Promise.resolved — for instantly-resolved values
override fun promiseReturnsInstantly(): Promise<Double> {
  return Promise.resolved(55.0)
}

// Promise.resolved() — for instantly-resolved void
override fun promiseThatResolvesVoidInstantly(): Promise<Unit> {
  return Promise.resolved()
}
```

### Accessing Android Context

Use `NitroModules.applicationContext` to get the `ReactApplicationContext` as it can be accessed anywhere in your hybrid objects. Always access it lazily via a property — never store it in a field, as it can be null during initialization.

```kotlin
import android.content.Context
import android.content.SharedPreferences
import com.margelo.nitro.NitroModules

@Keep
@DoNotStrip
class HybridStorage : HybridStorageSpec() {

  // Lazily access context — throws if not yet set
  private val context: ReactApplicationContext
    get() = NitroModules.applicationContext ?: throw Error("No ApplicationContext set!")

  // Use context to get system services or app storage
  private val sharedPreferences: SharedPreferences
    get() = context.getSharedPreferences("com.margelo.storage", Context.MODE_PRIVATE)

  override fun getString(key: String): String? {
    return sharedPreferences.getString(key, null)
  }

  override fun setString(key: String, value: String) {
    sharedPreferences.edit().putString(key, value).apply()
  }
}
```

**Rules:**
- Always use `get()` — never `val context = NitroModules.applicationContext` at class level, it may be null at construction time
- Always null-check with `?: throw Error(...)` so failures are explicit, not silent NPEs
- `ReactApplicationContext` is a subclass of Android `Context` — use it for `getSharedPreferences`, `getSystemService`, file access, etc.

### Using callbacks

```kotlin
override fun compute(input: Double, onResult: (Double) -> Unit) {
  val result = input * 2.0
  onResult(result)
}
```

### Handling nullable / optional

```kotlin
override var optionalValue: Double? = null

override fun processOptional(value: Double?): Double {
  return value ?: 0.0
}
```

### Throwing errors

```kotlin
override fun divide(a: Double, b: Double): Double {
  if (b == 0.0) throw IllegalArgumentException("Division by zero!")
  return a / b
}
```

## Common Pitfalls

- **Missing `@Keep` or `@DoNotStrip`** — The class will be removed in release builds, causing crashes
- **Wrong package name** — Must match `com.margelo.nitro.<androidNamespace>` from `nitro.json`
- **`Long` vs `ULong`** — TypeScript `bigint` with `uint64` maps to `ULong` not `Long`
- **Not overriding all abstract members** — Kotlin will fail to compile if any abstract member is missing
- **`Promise<void>` vs `Promise<Unit>`** — Kotlin uses `Unit`, not `void`. Always `Promise<Unit>` for void async methods
- **`Array<Double>` vs `DoubleArray`** — Number arrays use `DoubleArray` (primitive), other types use `Array<T>`
- **`Promise.async` vs `Promise.parallel`** — Use `async` for IO/coroutine work, `parallel` for CPU-bound sync work
- **Calling blocking code outside `Promise.async`** — Network calls, delay, etc. must be inside `Promise.async { }` (uses coroutines)
- **Storing `NitroModules.applicationContext` in a field** — It can be null at construction time; always access it via a `get()` property
- **Not null-checking `applicationContext`** — Always use `?: throw Error("No ApplicationContext set!")` to fail explicitly

## Related Skills

- [native-nitrogen-codegen.md](native-nitrogen-codegen.md) — Must generate specs before implementing
- [spec-nitro-json.md](spec-nitro-json.md) — Configure `"kotlin"` in autolinking
- [native-implement-swift.md](native-implement-swift.md) — iOS Swift counterpart
- [native-implement-cpp.md](native-implement-cpp.md) — C++ cross-platform alternative
