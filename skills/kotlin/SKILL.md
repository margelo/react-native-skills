---
name: kotlin
description: Design, implement, and review Kotlin and Android APIs. Use when working on .kt files, Kotlin nullability, sealed classes/interfaces, coroutines, Android threading, Java interop, or Kotlin-backed React Native Nitro Module implementations.
---

# Kotlin

Use this skill for Kotlin code that needs strong type modeling, explicit async boundaries, and idiomatic Android integration. When the code is part of a Nitro Module, pair this with `build-nitro-modules` for generated specs, Promise mapping, annotations, and HybridObject constraints.

## Workflow

1. Read the local Kotlin code, generated abstract classes, and public API shape before editing.
2. Model valid states with Kotlin types before implementing behavior.
3. Choose whether work is synchronous, coroutine-based, or explicitly dispatched.
4. Keep properties cheap and nonblocking.
5. Make thread hops, Android service access, I/O, and fallible work visible in the API.

## Type-Safe API Design

- Represent state variants with sealed interfaces/classes, normal interfaces, or concrete classes. Do not encode variants as one data class with many nullable fields.
- Keep related values non-null on the variant where they are valid. If `barcode` and `barcodeType` must exist together, put them on `ScannedBarcode`.
- Use nullable types for real absence inside one state, not for "maybe this object is a different kind of thing".
- Prefer compile-time narrowing over caller-side null probing. Repeated null checks for related fields usually mean the model is too loose.

```kotlin
// Avoid: related fields are nullable and the valid combinations are implicit.
data class ScannedData(
  val position: Point,
  val text: String?,
  val barcode: String?,
  val barcodeType: BarcodeType?,
  val face: Rect?,
)

// Prefer: each variant exposes only the fields that are valid for it.
sealed interface ScannedData {
  val position: Point
}

data class ScannedText(
  override val position: Point,
  val text: String,
) : ScannedData

data class ScannedBarcode(
  override val position: Point,
  val barcode: String,
  val barcodeType: BarcodeType,
) : ScannedData

data class ScannedFace(
  override val position: Point,
  val face: Rect,
) : ScannedData
```

## Async and Threading

- Use coroutines for naturally suspending APIs and Android I/O that already has coroutine support.
- Use explicit dispatchers or Nitro `Promise.parallel` for CPU-bound synchronous work that should not run on the caller thread.
- Do not use `runBlocking` in library code, property getters, setters, or JS-facing entry points.
- Do not hide a thread hop, service lookup, blocking I/O, permission flow, or fallible native operation behind a property.
- If a value is emitted by a specific Android callback/thread, prefer a listener, Flow, or event API. In Nitro specs, expose a listener or a `Promise<T>` method.
- Keep mutable shared state owned by one coroutine scope, dispatcher, lock, or Android component lifecycle. Avoid mixing ownership models without a clear boundary.

## Properties

- Use `val` and `var` for cheap in-memory state or derived values.
- Use methods for side effects, I/O, Android service calls, permission checks, allocation, blocking work, or operations that can fail.
- If a property would need to wait for another thread, redesign it as an async method or event.

```kotlin
// Avoid: blocking work hidden in a getter.
val status: SessionStatus
  get() = runBlocking { session.status() }

// Prefer: make the async boundary explicit.
fun getStatus(): Promise<SessionStatus> {
  return Promise.async {
    session.status()
  }
}
```

## Kotlin Style

- Make classes final by default. Kotlin classes already are final unless marked `open`; keep them that way unless inheritance is intentional.
- Prefer `data class` for plain values and regular classes for owned resources or lifecycle.
- Prefer `sealed interface` or `sealed class` for closed result families.
- Use `require`, `check`, or specific exceptions for invalid inputs and invalid state. Do not silently no-op user-reachable failures.
- Keep Java interop explicit. Convert platform types into Kotlin types before exposing them through the public API where practical.
- Avoid `!!` except at narrow boundaries where a prior check makes the invariant obvious. Prefer early returns, `requireNotNull`, or typed state.

## Nitro Notes

- Kotlin HybridObject implementations must extend the generated `Hybrid*Spec` class and include `@Keep` plus `@DoNotStrip`.
- Use `Promise.async` for suspending or I/O work and `Promise.parallel` for CPU-bound synchronous work.
- Use `Promise<Unit>` for `Promise<void>`.
- Access `NitroModules.applicationContext` lazily and fail explicitly if it is unavailable.
- Generated properties are synchronous JS entry points. Redesign them as methods or listeners if they need Android thread affinity or async work.
