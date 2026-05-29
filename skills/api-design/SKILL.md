---
name: api-design
description: Design and review predictable public APIs for TypeScript, JavaScript, React, and React Native libraries. Use when shaping exported functions, classes, hooks, options objects, event and listener APIs, error behavior, naming, cross-platform abstractions, or JS-only packages. Pair with build-nitro-modules when the library is backed by Nitro.
---

# API Design

Use this skill before implementation or when reviewing a public API surface. Optimize for APIs that are explicit, typed, predictable, and easy to evolve without forcing users to learn irrelevant internals.

If the library is a Nitro Module, use this skill for the public TypeScript and React API shape first, then use `build-nitro-modules` for Nitro-specific spec, native-state, and binding constraints. If the library is JS-only, React-only, or React Native JS-only, stay in this skill.

## Workflow

1. Sketch the user-facing TypeScript API before implementing internals.
2. Write 2-3 realistic call-site examples, including error and cleanup paths.
3. Check the surface against the rules below.
4. Implement only after the public shape is coherent.

## API Shape Rules

- Prefer a single source of truth. Do not split related state across booleans and dependent values when one typed value can express the state. Prefer `timeoutMs?: number` over `enableTimeout: boolean` plus `timeoutMs?: number`.
- Use option objects or named structs once a function has 3 or more parameters, parameters of the same primitive type, or values that are likely to grow.
- Keep APIs specific instead of accepting every possible input shape. A millisecond timeout should be a `number`, not `number | string | bigint | object | null`.
- Avoid giant "does everything" objects. Split by domain or lifecycle when responsibilities differ.
- Prefer literal unions, enums, discriminated unions, interfaces, and typed option groups when the valid states are known.
- Avoid untyped dictionaries, boolean clusters, stringly typed commands, and loosely shaped events when the valid states are known.
- Use `undefined` or optional fields for absence. Use `null` only when "explicit none" means something different from "not provided".
- Prefer discriminated unions for state machines, loading states, and result variants.

## Names and Members

- Use descriptive names that include the action, subject, and unit when useful. Prefer `getCurrentSystemTimestampMs()` over `timestamp()`.
- Use `is*`, `has*`, `can*`, or similar prefixes for boolean properties.
- Use properties for cheap observed state or capability. Prefer `readonly isAccelerometerAvailable: boolean` over `accelerometerAvailable()`.
- Use methods for side effects, expensive work, allocation, mutation, async boundaries, or operations that can fail.
- Make units explicit in names: `timeoutMs`, `byteSize`, `maxRetries`, `createdAt`.

## Async, Events, and React

- Use sync APIs only when results are immediate, deterministic, and cheap.
- Use `Promise` for one-shot async work. Use listener APIs, streams, or observable stores for repeated events.
- Prefer explicit listener methods with cleanup over writable callback properties: `addListener(listener): Subscription` is easier to reason about than `onEvent?: (...) => void`.
- In React or React Native libraries, provide hooks on top of a stable imperative API when the value should drive UI. Use the appropriate React primitive, such as `useEffect` for subscriptions or `useSyncExternalStore` for external state.
- Keep hooks as adapters over the imperative core, not the only way to use the library.

## Errors and Platform Differences

- Never silently swallow user-reachable errors. Do not just return, log, or no-op. Throw, reject, or emit through a dedicated error listener.
- Unsupported capabilities or platforms should throw specific errors. Name the missing capability, invalid value, valid range, or platform restriction.
- Mention an alternative in the error message when a real alternative exists.
- Static assertions or fatal errors are only for impossible internal states, not for states reachable from JS user code.
- For React Native, design cross-platform concepts instead of mirroring iOS and Android APIs 1:1. Avoid leaking native class names or framework details unless that low-level access is the feature.
- Do not over-abstract into a generic web-shaped API if the value is closer to the metal, such as GPU, zero-copy, native-resource, or hardware concepts.

## TypeScript Facades

- Put defaults, convenience overloads, and ergonomic discriminated unions in TypeScript when that keeps the lower-level implementation simpler and explicit.
- Keep TypeScript adapters thin. If the facade starts hiding important behavior or adding inconsistent semantics, move the rule into the core API instead.
- Prefer examples that show realistic setup, cleanup, unavailable-feature behavior, and invalid inputs.
