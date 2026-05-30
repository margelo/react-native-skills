---
name: api-design
description: Design and review predictable public APIs for TypeScript, JavaScript, React, and React Native libraries. Use when shaping exported functions, classes, hooks, options objects, event and listener APIs, error behavior, naming, cross-platform abstractions, or JS-only packages. Pair with build-nitro-modules when the library is backed by Nitro.
---

# API Design

Use this skill before implementation or when reviewing a public API surface. Target explicit types, stable semantics, and no irrelevant internals in the public contract.

If the library is a Nitro Module, use this skill for the public TypeScript and React API shape first, then use `build-nitro-modules` for Nitro-specific spec, native-state, and binding constraints. If the library is JS-only, React-only, or React Native JS-only, stay in this skill.

## Workflow

1. Sketch the user-facing TypeScript API before implementing internals.
2. Write 2-3 realistic call-site examples, including error and cleanup paths.
3. Check the surface against the rules below.
4. Implement only after the public shape is coherent.

## API Freshness

Before choosing public API shape, dependency APIs, platform capabilities, or implementation strategy, verify current official sources instead of relying on trained memory. Library, React, React Native, platform, and tooling APIs evolve quickly.

- Prefer official docs, source repositories, release notes, changelogs, package READMEs, and current package metadata.
- Look for `llms.txt` or `llms-full.txt` on official docs sites when available, and use those as compact current context.
- Treat remembered API details as a starting hypothesis only. If current docs or source disagree, follow the current docs/source and mention the change when relevant.
- Avoid designing against stale blog posts, old snippets, or outdated trained assumptions when an official current source is available.

## API Shape Rules

- Prefer a single source of truth. Do not split related state across booleans and dependent values when one typed value can express the state. Prefer `timeoutMs?: number` over `enableTimeout: boolean` plus `timeoutMs?: number`.
- Use option objects or named structs once a function has 3 or more parameters, parameters of the same primitive type, or values that are likely to grow.
- Keep APIs specific instead of accepting every possible input shape. A millisecond timeout should be a `number`, not `number | string | bigint | object | null`.
- Avoid giant "does everything" objects. Split by domain or lifecycle when responsibilities differ.
- Prefer literal unions, discriminated unions, interfaces, and typed option groups when the valid states are known. In TypeScript libraries, prefer string literal unions over runtime `enum`s unless consumers need a runtime value.
- Avoid untyped dictionaries, boolean clusters, stringly typed commands, and loosely shaped events when the valid states are known.
- Use `undefined` or optional fields for absence. Use `null` only when "explicit none" means something different from "not provided".
- Prefer discriminated unions for state machines, loading states, and result variants.
- Model user intent separately from resolved state when negotiation is involved. For example, an ordered array of constraint objects can express priorities, while a resolved config object reports what the platform actually selected.
- Expose common presets as `as const satisfies Record<string, Type>` objects. Keep the accepted type structural so users can provide their own values.
- Do not expose ambient facts the caller already knows, such as a `platform` field that only repeats `Platform.OS`, unless the API can return data produced by a different platform than the current runtime.
- Do not freeze today's platform support matrix into the type shape. Prefer runtime capability fields such as `availableTextTypes: []` or `supportedFormats: []` over separate platform-specific types or static exclusions. This lets newer native capabilities become available without redesigning the JS API.
- Prefer one unified options object. Avoid `ios`/`android` option bags and platform-prefixed methods unless the concepts are genuinely platform-only and cannot be described as a cross-platform capability or no-op.
- Do not return half-initialized objects that require a separate `prepare()`, `initialize()`, or `load()` call before normal use. If setup is required, make the factory async and resolve with a ready object. Keep lifecycle methods for real repeatable transitions such as `start()`/`stop()`, not construction readiness.
- For larger libraries, expose one small public root or factory that creates stateful domain objects. Keep object construction, async setup, I/O, and validation behind factory methods instead of forcing callers through static functions or half-ready instances.
- Return `undefined` only for normal domain absence, and document exactly when it occurs. Do not use optional returns as an unstated error path; throw or reject when an operation fails.

## Public API Organization

- Split public surfaces into focused files or modules and re-export them from a clear package entry point. Do not create catch-all files that contain a feature's main object plus every enum, option, result, event, and helper type.
- Default to one exported public type per file. Group multiple exported types in one file only when they form one tightly coupled logical construct and are rarely imported independently, such as a `DynamicRange` type plus the exact literal unions that define it.
- A 600-line type/spec file is not acceptable when the types can be split by concept, such as barcode formats, scanned values, configuration, capabilities, and subscriptions.
- Put shared fields and methods on a base interface or class. Subtypes should add only the members that are specific to that subtype, not repeat fields such as `format`, `valueType`, `id`, or `bounds` across every concrete variant.
- Keep performance and implementation strategy out of the public shape unless it changes how the caller should use the API. For example, the API may expose an explicit conversion method, but names and docs should not advertise internal details like "lazy", "lightweight", or "normalized" unless that behavior is directly observable.

## Names and Members

- Name commands with a verb and subject. Add unit suffixes to numeric values. Prefer `getCurrentSystemTimestampMs()` over `timestamp()`.
- Use `is*`, `has*`, or `can*` prefixes for boolean properties.
- Use properties for cheap observed state or capability. Prefer `readonly isAccelerometerAvailable: boolean` over `accelerometerAvailable()`.
- Use methods for side effects, expensive work, allocation, mutation, async boundaries, or operations that can fail.
- Make units explicit in names: `timeoutMs`, `byteSize`, `maxRetries`, `createdAt`.
- Avoid mechanically prefixing every exported type with the package, module, or feature name. Imports and package namespaces already provide context, and callers can alias names when needed. Use prefixes only when the unprefixed name is ambiguous at package-root import sites; prefer domain names like `CameraPermissionStatus`, `ScannedResultSource`, `Point`, or `Rect` over repeating `DataScanner` on every type.
- Use `readonly` for public state the caller observes but does not set. Use mutable properties only when assigning the property is itself the intended command.
- Writable properties must be cheap, synchronous, and unlikely to fail. If setting a value needs native negotiation, permissions, I/O, allocation, fallible validation, or thread/process hops, expose an explicit async method such as `setZoomFactor(zoomFactor): Promise<void>`.
- For public string literal unions, prefer lowercase kebab-case values such as `manual-input` over camelCase values such as `manualInput`.

## Async, Events, and React

- Use sync APIs only when results are immediate, deterministic, and cheap.
- Use `Promise` for one-shot async work. Use listener APIs, streams, or observable stores for repeated events.
- Use sync methods only for fast in-process work, cheap object creation, cached metadata, and local transforms. Use `Promise` for permissions, hardware/session setup, I/O, capture/recording, platform async APIs, blocking work, or work that may cross a thread or process boundary.
- Benchmark ambiguous APIs. Make the method async if it takes longer than roughly 50ms in normal use.
- If a transform has both cheap and potentially heavy paths, expose explicit sync and async methods such as `convertX()` and `convertXAsync()` instead of hiding blocking work behind one ambiguous method.
- Use explicit listener methods with cleanup instead of writable callback properties: `addListener(listener): Subscription`, not `onEvent?: (...) => void`.
- Listener registration must return the cleanup handle. Avoid `addListener(...): number` plus `removeListener(listenerId: number)` because caller-managed IDs are easy to leak, hard to type safely, and force hidden global/static listener tables. Prefer a flat subscription object with an idempotent `remove(): void` method that owns the native or internal cleanup.
- Use callback option objects for one-shot operation progress when callbacks belong to one method call, such as capture or upload progress callbacks.
- Do not expose native-side bookkeeping as the event contract unless it is the feature. Prefer simple observations and let JS derive app-specific diffs or state when that is cheap and clearer.
- In React or React Native libraries, provide hooks on top of a stable imperative API when the value should drive UI. Use the appropriate React primitive, such as `useEffect` for subscriptions or `useSyncExternalStore` for external state.
- Keep hooks as adapters over the imperative core, not the only way to use the library.
- For React components, wrap the imperative core instead of creating a parallel API. A component can own setup, refs, gestures, and lifecycle, while hooks expose the same lower-level objects for advanced use.
- Use stable callbacks and deterministic cleanup for subscriptions. Subscribe before lifecycle effects that can emit events when missing an event would be observable.

## Errors and Platform Differences

- Never silently swallow user-reachable errors. Do not just return, log, or no-op. Throw, reject, or emit through a dedicated error listener.
- Use real JavaScript `Error` objects for failures and error events, not custom `{ code, message }` structs, unless the API is deliberately serializing errors across a boundary that cannot preserve prototypes.
- Unsupported capabilities or platforms should throw specific errors. Name the missing capability, invalid value, valid range, or platform restriction.
- Mention an alternative in the error message when a real alternative exists.
- Static assertions or fatal errors are only for impossible internal states, not for states reachable from JS user code.
- For React Native, design cross-platform concepts instead of mirroring iOS and Android APIs 1:1. Avoid leaking native class names or framework details unless that low-level access is the feature.
- Keep low-level concepts explicit for GPU, zero-copy, native-resource, and hardware APIs. Do not force them into generic web-shaped names if that hides ownership, copying, sync/async behavior, thread affinity, or disposal.

## Avoid Workaround APIs

- Do not encode temporary patches, platform bugs, dependency quirks, or stale implementation details into the public API shape unless callers genuinely need to reason about them.
- Prefer fixing the root cause, using official extension points, or hiding compatibility code behind a stable domain API.
- If a workaround is unavoidable, keep it narrow and internal where possible. If it must be public, document why it exists, what issue it tracks, and when it can be removed.

## TypeScript Facades

- Put defaults, convenience overloads, and discriminated unions in TypeScript only when they reduce implementation branches without changing behavior.
- Keep TypeScript adapters thin. If the facade changes failure behavior, defaults, lifecycle, or ownership semantics, move the rule into the core API instead.
- Prefer examples that show realistic setup, cleanup, unavailable-feature behavior, and invalid inputs.

## Documentation Contracts

- Treat exported TypeScript as product documentation. Add JSDoc to every exported type-level declaration, including interfaces, type aliases, string-literal unions, runtime enums, classes, HybridObjects, callbacks, option objects, event objects, hooks, components, and public constants.
- Use JSDoc for semantics, lifecycle, platform behavior, defaults, and performance costs that types cannot express.
- Keep JSDoc user-facing. Do not mention native class names, framework implementation details, or current platform limitations unless the caller must know them to use the API correctly.
- Write JSDoc that explains the domain meaning, not the fact that the API is JavaScript. Avoid empty comments such as "normalized for JavaScript"; prefer concrete semantics such as "Represents the format of a barcode" or "Represents the content type of a scanned text value."
- Document every public property, including properties on options, capabilities, results, events, and HybridObjects. Short comments are fine for obvious fields, but defaults, units, availability, failure behavior, and interaction with related options belong on the property that exposes them.
- Every type-level JSDoc should connect the type to a real related API with `{@linkcode ...}` or `@see`. Use the property, method, factory, or result that exposes the type, for example `Represents the format of a {@linkcode Barcode}.` plus `@see {@linkcode Barcode.format}`. Do not invent link targets or add filler links only to satisfy this rule.
- For base interfaces or abstract concepts, describe how callers encounter the value and link concrete variants exported by the package, for example `Represents a value scanned by {@linkcode DataScanner}. Concrete values include {@linkcode ScannedTextValue} and {@linkcode ScannedBarcodeValue}.`
- Use `{@linkcode ...}` or `@see` to point to related methods, configuration objects, and capability checks instead of writing generic warnings about not assuming the current OS or platform.
- Use `@default`, `@throws`, `@platform`, `@example`, `@see`, and `@discussion` where they clarify behavior. Link related APIs with `{@linkcode ...}`.
- Document resource ownership and cleanup explicitly. If a returned object must be disposed, say when and why.
