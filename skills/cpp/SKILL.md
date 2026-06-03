---
name: cpp
description: Design, implement, and review modern C++ APIs and native implementation code. Use when working on .cpp, .hpp, CMake, RAII ownership, std::variant, callbacks, async work, platform bridges, or C++-backed React Native Nitro Module implementations.
---

# C++

Use this skill for C++ code that needs clear ownership, strong type modeling, maintainable file boundaries, and predictable native integration. When the code is part of a Nitro Module, pair this with `build-nitro-modules` for generated specs, Promise mapping, CMake, and HybridObject constraints.

## Workflow

1. Read the generated headers, public API shape, and local ownership patterns before editing.
2. Model valid states with types before implementing behavior.
3. Keep headers small and stable; push implementation details into `.cpp` files.
4. Split helpers by responsibility instead of growing a primary implementation file.
5. Make ownership, lifetime, threading, and async boundaries explicit.

## Type-Safe API Design

- Represent state variants with `std::variant`, inheritance, or separate concrete types. Do not encode variants as one struct with many `std::optional` fields.
- Keep related values nonoptional on the variant where they are valid. If `barcode` and `barcodeType` must exist together, put both on `ScannedBarcode`.
- Use `std::optional` for real domain absence inside one state, not for "maybe this object is a different kind of thing".
- Prefer compile-time flow over caller-side optional probing.

## File Organization

- Treat a filename as a scope contract. `HybridDataScanner.cpp` should implement `HybridDataScanner`; it should not also define unrelated geometry conversions, UI helpers, platform adapters, or extension-style utilities.
- Keep one primary class, struct family, or cohesive algorithm per file by default. Split delegates, adapters, converters, platform shims, and reusable helpers into named files such as `GeometryConversions.cpp`, `DataScannerDelegate.cpp`, or `PlatformViewController.cpp`.
- Put private helpers in an anonymous namespace only when they are small and only serve the file's primary type. Move reusable or bulky helpers to a separate file under a `detail` namespace or an internal folder.
- Use line count as a review signal: files below roughly 300 lines are usually fine; files above that need a clear reason tied to one cohesive responsibility. Size caused by unrelated helpers, conversions, or platform glue is a design issue.
- Do not hide large implementation bodies in headers. Use headers for declarations, templates that must be visible, and small inline functions only.
- Put conversions on the element type or as a named converter function for one element when the batch operation is only a standard loop or transform. Prefer `toNativeRecognizedDataType(type)` plus `std::transform` over a domain-specific vector helper that only wraps the transform.
- Add collection-level helpers only when the collection has real domain behavior, such as validation across elements, deduplication, ordering, batching, caching, or error aggregation.

## Ownership and Async

- Prefer RAII and explicit ownership with values, `std::unique_ptr`, and `std::shared_ptr` only when shared lifetime is real.
- Avoid raw owning pointers. Raw pointers and references should be non-owning and documented by surrounding lifetime.
- Use `std::variant`, `std::optional`, and strong structs to express API flow instead of stringly typed states.
- Do not block caller threads for I/O, long CPU work, or platform callbacks. Use the project's async abstraction, executor, or Nitro `Promise<T>` when the operation can wait.
- Keep thread-affine state behind a clear owner. Do not mix mutexes, queues, callbacks, and shared ownership without a documented boundary.

## Nitro Notes

- C++ HybridObjects must inherit from the generated `Hybrid*Spec` class and copy generated signatures exactly.
- Use `std::shared_ptr<Promise<T>>` for generated async methods.
- Keep generated-spec implementation files focused on the HybridObject. Put conversion helpers, OpenCV utilities, platform adapters, and CMake-only glue in separate files.
- Pair JS-facing base HybridObjects with native interfaces or adapter classes when C++ needs to work with Swift/Kotlin-backed platform services.
