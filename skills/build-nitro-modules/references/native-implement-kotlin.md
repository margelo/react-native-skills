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

- Nitrogen has generated `HybridMathSpec.kt`, usually in `nitrogen/generated/android/kotlin/com/margelo/nitro/<namespace>/`
- `nitro.json` has an Android autolinking entry with `"language": "kotlin"` and `"implementationClassName": "HybridMath"`

## Step-by-Step

### 1. Locate the generated spec

```
nitrogen/generated/android/kotlin/com/margelo/nitro/math/HybridMathSpec.kt
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

`@DoNotStrip` is required when ProGuard/R8 can strip the class. Keep `@Keep` as well to match generated Nitro code and common library implementations.

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

Use the Promise helper that matches the work:
- `Promise.async` for suspending or I/O work that should run through coroutines.
- `Promise.parallel` for CPU-bound synchronous work that should run off the caller thread.
- Avoid manually creating or passing around `Promise<T>` instances by default. Prefer `Promise.async`, `Promise.parallel`, `Promise.resolved`, or `Promise.rejected` so the helper owns exactly-once completion. A manual Promise is only justified when bridging a native completion/listener/callback API; keep it near the bridge, do not pass it through arbitrary helpers or session objects, and make every branch resolve or reject exactly once.
- Do not use `runBlocking` in HybridObject methods, generated property getters/setters, or library callbacks. If callers must wait for a result, expose a `Promise<T>` method in the Nitro spec.
- Treat repeated `launch`, `withContext`, dispatcher, handler, executor, or JS/Nitro runtime hops as an architecture smell. A HybridObject/session should own the coroutine scope/dispatcher/lifecycle it works on, or cross into that owner once at the Promise, lifecycle, or native callback boundary.
- If an operation bounces between main, IO, default, native, and JS/Nitro contexts in multiple nested places, redesign the HybridObject boundary or returned lifecycle handle instead of adding more hops.
- Do not fix lifecycle, readiness, or race bugs with `delay`, `Thread.sleep`, `Handler.postDelayed`, timers, extra dispatcher hops, or calling the same native method twice. Use an explicit Promise, callback, listener event, Flow, returned configured HybridObject, or state transition instead. Retry only for external hardware, OS service, remote service, or network uncertainty, with bounded/cancellable/idempotent behavior.
- Treat `Mutex`, `synchronized`, and `ReentrantLock` as last-resort synchronization. Before adding one to a HybridObject, identify the concrete shared mutable values, the threads/dispatchers that can access them concurrently, and why one owner dispatcher/lifecycle, message passing, or immutable snapshots is not enough.
- Never invoke JS/Nitro callbacks while holding a lock. Snapshot listener collections if needed, unlock, then call them. A listener removed during an in-flight emission may receive that current event.

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

Use Kotlin nullable types for real domain absence, not for modeling several possible object states. If fields are related, put them on a non-null variant type in the TypeScript spec and the generated Kotlin implementation.

```kotlin
override var optionalValue: Double? = null

override fun processOptional(value: Double?): Double {
  return value ?: 0.0
}
```

For closed Kotlin-only helper state, prefer `sealed interface` or `sealed class` over a data class with many nullable fields.

### Properties and thread affinity

Generated Nitro properties are synchronous JS entry points. Keep `val`/`var` access cheap and local. Do not hide blocking work, `runBlocking`, Android service calls, permission flows, or thread hops in a getter or setter.

```kotlin
// Avoid: blocking async work hidden in a getter.
val status: SessionStatus
  get() = runBlocking { session.status() }

// Prefer: make the async boundary explicit in the Nitro spec.
override fun getStatus(): Promise<SessionStatus> {
  return Promise.async {
    session.status()
  }
}
```

### Throwing errors

```kotlin
override fun divide(a: Double, b: Double): Double {
  if (b == 0.0) throw IllegalArgumentException("Division by zero!")
  return a / b
}
```

### Validating configuration

Validate invalid values and required behavior early, but do not reject optional cross-platform preferences only because Android ignores them. If the operation still does its core job, ignore or degrade the preference and report support through the public capabilities or resolved state.

Examples:
- Throw when a requested flash mode cannot work because the device has no flash.
- Do not throw only because a quality, guidance UI, high-frame-rate, auto-zoom, or region preference is unsupported, unless the API documents that field as a hard requirement.

### Kotlin style and organization

- Treat `HybridDataScanner.kt` as the implementation file for `HybridDataScanner`, not as a dumping ground for unrelated Android helpers, geometry conversions, listeners, or extension utilities.
- Keep one primary class, sealed family, or cohesive extension family per file by default.
- Move reusable extensions, Android adapters, conversion helpers, delegates/listeners, and protocol-style interfaces into named files such as `Extensions/ViewExtensions.kt`, `Conversions/PointConversions.kt`, or `DataScannerDelegate.kt`.
- Use `internal` visibility for helpers that should stay inside the module.
- Use line count as a review signal: under roughly 300 lines is usually acceptable, while files above that need a concrete reason tied to one cohesive responsibility. A large file caused by helpers or Android glue belongs in multiple files.
- Put one-element conversions on the source type. The element method may return a list/set when one source value expands to several native values.
- Compose collections where they are used with `map`, `flatMap`, folds, or sets. Keep an aggregate helper only when it owns real collection semantics such as deduplication, validation across elements, nonempty checks, batching, caching, or error aggregation, and place it in a focused conversion/configuration file rather than the main HybridObject file.

## Common Pitfalls

- **Missing `@Keep` or `@DoNotStrip`** — The class will be removed in release builds, causing crashes
- **Wrong package name** — Must match `com.margelo.nitro.<androidNamespace>` from `nitro.json`
- **`Long` vs `ULong`** — TypeScript `bigint` with `uint64` maps to `ULong` not `Long`
- **Not overriding all abstract members** — Kotlin will fail to compile if any abstract member is missing
- **`Promise<void>` vs `Promise<Unit>`** — Kotlin uses `Unit`, not `void`. Always `Promise<Unit>` for void async methods
- **`Array<Double>` vs `DoubleArray`** — Number arrays use `DoubleArray` (primitive), other types use `Array<T>`
- **`Promise.async` vs `Promise.parallel`** — Use `async` for IO/coroutine work, `parallel` for CPU-bound sync work
- **Calling blocking code outside `Promise.async`** — Network calls, delay, etc. must be inside `Promise.async { }` (uses coroutines)
- **Using `runBlocking` in generated entry points** — Do not block JS-facing methods or properties; expose a `Promise<T>` method or listener instead
- **Modeling variants with nullable clusters** — Use distinct TypeScript/Nitro variants so Kotlin receives non-null related fields
- **Storing `NitroModules.applicationContext` in a field** — It can be null at construction time; always access it via a `get()` property
- **Not null-checking `applicationContext`** — Always use `?: throw Error("No ApplicationContext set!")` to fail explicitly
- **Letting one HybridObject file absorb every helper** — Split extensions, adapters, converters, and listeners into named files. The filename should still describe the file after the implementation is done.
- **Putting trivial maps behind collection extensions** — Prefer an element conversion plus `map`/`flatMap` at the call site. A collection helper is justified only when the collection itself adds behavior such as deduplication or validation.

## Related Skills

- [kotlin](../../kotlin/SKILL.md) — General Kotlin API, nullability, coroutine, and threading guidance
- [native-nitrogen-codegen.md](native-nitrogen-codegen.md) — Must generate specs before implementing
- [spec-nitro-json.md](spec-nitro-json.md) — Configure `"kotlin"` in autolinking
- [native-implement-swift.md](native-implement-swift.md) — iOS Swift counterpart
- [native-implement-cpp.md](native-implement-cpp.md) — C++ cross-platform alternative
