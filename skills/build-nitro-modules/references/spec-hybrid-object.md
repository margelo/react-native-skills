---
title: Writing HybridObject Specs and Exporting
impact: CRITICAL
tags: hybrid-object, spec, nitro.ts, typescript, NitroModules, export, interface
---

# Skill: Writing HybridObject Specs and Exporting

Covers Steps 4 and 10: deleting the default spec, writing a domain-specific `*.nitro.ts` spec, and exporting the HybridObject.

## Quick Pattern

**Incorrect** — keeping the default stub:
```typescript
// src/specs/Example.nitro.ts  ← DELETE THIS
import { type HybridObject } from 'react-native-nitro-modules'
interface Example extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {}
```

**Correct** — domain-specific spec with export:
```typescript
// src/specs/Math.nitro.ts
import { type HybridObject, NitroModules } from 'react-native-nitro-modules'

interface Math extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  add(a: number, b: number): number
  subtract(a: number, b: number): number
  multiply(a: number, b: number): Promise<number>
}

const math = NitroModules.createHybridObject<Math>('Math')
export { math }
```

## When to Use

- After scaffolding with `nitrogen init`, before running `npx nitrogen`
- After any API change to the module's interface
- When adding new methods or properties to an existing module

## Prerequisites

- Library scaffolded via `npx nitrogen@latest init <name>`
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

### 3. Write the interface

```typescript
import { type HybridObject, NitroModules } from 'react-native-nitro-modules'

interface Math extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
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

### 5. Export the HybridObject (Step 10)

After implementing native code, export from `src/index.ts`:

```typescript
// src/index.ts
import { NitroModules } from 'react-native-nitro-modules'
import type { Math } from './specs/Math.nitro'

const math = NitroModules.createHybridObject<Math>('Math')

export { math }
export type { Math }
```

**Rules:**
- The string `'Math'` in `createHybridObject<Math>('Math')` must exactly match the key in `nitro.json`'s `autolinking` block
- Prefer naming native classes with the `Hybrid` prefix: `HybridMath`
- Keep both the interface name and the autolinking key the same (e.g. `Math` = `'Math'`)

## Code Examples

### Module with properties and callbacks

```typescript
import { type HybridObject, NitroModules } from 'react-native-nitro-modules'

interface Camera extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  // Properties
  readonly isRecording: boolean
  zoom: number

  // Callbacks
  onFrameCaptured: ((frame: ArrayBuffer) => void) | undefined

  // Async methods
  startRecording(): Promise<void>
  stopRecording(): Promise<string>  // returns file path

  // Sync methods
  setFlashMode(mode: 'on' | 'off' | 'auto'): void
}

const camera = NitroModules.createHybridObject<Camera>('Camera')
export { camera }
```

### TypeScript → Native type mapping

| TypeScript | C++ | Kotlin | Swift |
|-----------|-----|--------|-------|
| `number` | `double` | `Double` | `Double` |
| `string` | `std::string` | `String` | `String` |
| `boolean` | `bool` | `Boolean` | `Bool` |
| `bigint` | `int64_t` / `uint64_t` | `Long` / `ULong` | `Int64` / `UInt64` |
| `T[]` | `std::vector<T>` | `Array<T>` | `[T]` |
| `Promise<T>` | `std::future<T>` | `Promise<T>` | `Promise<T>` |
| `T \| undefined` | `std::optional<T>` | `T?` | `T?` |
| `(x: T) => void` | `std::function<void(T)>` | `(T) -> Unit` | `(T) -> Void` |
| `ArrayBuffer` | `std::shared_ptr<ArrayBuffer>` | `ArrayBuffer` | `ArrayBuffer` |

## Common Pitfalls

- **Wrong file extension** — Must be `.nitro.ts`, not `.ts` or `.d.ts`
- **Mismatch between interface name and autolinking key** — `createHybridObject<Math>('Math')` string must match `nitro.json`
- **Forgetting platform languages** — `HybridObject<{}>` without specifying ios/android will fail
- **Modifying generated files** — Never edit files in `nitrogen/generated/`; edit only the `.nitro.ts` spec
- **Missing export** — The hybrid object won't be usable without the `createHybridObject` call and export

## Related Skills

- [spec-nitro-json.md](spec-nitro-json.md) — Configure autolinking to match the interface name
- [native-nitrogen-codegen.md](native-nitrogen-codegen.md) — Run codegen after writing the spec
