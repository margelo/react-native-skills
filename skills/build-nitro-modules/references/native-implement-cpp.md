---
title: Implementing HybridObjects in C++
impact: HIGH
tags: c++, hybrid-object, native, implementation, cpp, cross-platform, namespace, override
---

# Skill: Implementing HybridObjects in C++

Covers Steps 8–9 (C++ path): creating the C++ implementation class that inherits from the Nitrogen-generated spec.

## Quick Pattern

**Incorrect** — modifying generated files:
```cpp
// nitrogen/generated/shared/HybridMathSpec.hpp ← NEVER EDIT
```

**Correct** — implementing in a separate file:
```cpp
// cpp/HybridMath.hpp
#pragma once
#include "HybridMathSpec.hpp"

namespace margelo::nitro::math {
  class HybridMath: public HybridMathSpec {
  public:
    HybridMath() : HybridObject(TAG) {}
    double add(double a, double b) override;
  };
}
```

## When to Use

- When the spec uses `{ ios: 'cpp'; android: 'cpp' }` (shared C++ implementation)
- When implementing platform-agnostic logic that runs on both iOS and Android
- When performance or code-sharing across platforms is critical

## Prerequisites

- Nitrogen has generated `HybridMathSpec.hpp` in `nitrogen/generated/shared/`
- `nitro.json` has `"all": { "language": "c++", "implementationClassName": "HybridMath" }` in the autolinking block

## Step-by-Step

### 1. Locate the generated spec

```
nitrogen/generated/shared/HybridMathSpec.hpp   ← abstract base class
```

### 2. Create the implementation header

```bash
touch cpp/HybridMath.hpp
```

```cpp
// cpp/HybridMath.hpp
#pragma once
#include "HybridMathSpec.hpp"

namespace margelo::nitro::math {

  class HybridMath: public HybridMathSpec {
  public:
    HybridMath() : HybridObject(TAG) {}

  public:
    // Implement all pure virtual methods from the generated spec
    double add(double a, double b) override;
    double subtract(double a, double b) override;
    std::shared_ptr<Promise<double>> calculateFibonacci(double n) override;

    // Properties
    double getPi() override;
    double getPrecision() override;
    void setPrecision(double precision) override;

  private:
    double _precision = 6.0;

  public:
    inline static const char* TAG = "Math";
  };

} // namespace margelo::nitro::math
```

### 3. Create the implementation source file

```bash
touch cpp/HybridMath.cpp
```

```cpp
// cpp/HybridMath.cpp
#include "HybridMath.hpp"

namespace margelo::nitro::math {

  double HybridMath::add(double a, double b) {
    return a + b;
  }

  double HybridMath::subtract(double a, double b) {
    return a - b;
  }

  std::shared_ptr<Promise<double>> HybridMath::calculateFibonacci(double n) {
    return Promise<double>::async([n]() -> double {
      if (n <= 1) return n;
      double a = 0, b = 1;
      for (int i = 2; i <= n; i++) {
        double temp = a + b;
        a = b;
        b = temp;
      }
      return b;
    });
  }

  double HybridMath::getPi() {
    return M_PI;
  }

  double HybridMath::getPrecision() {
    return _precision;
  }

  void HybridMath::setPrecision(double precision) {
    _precision = precision;
  }

} // namespace margelo::nitro::math
```

### 4. Register in CMakeLists.txt

Add the implementation file to `android/CMakeLists.txt`:

```cmake
add_library(
  ReactNativeMath
  SHARED
  ../nitrogen/generated/shared/NitroMathSpecs.cpp
  ../cpp/HybridMath.cpp   # ← add this
)
```

### 5. Verify using canonical type reference

For any type uncertainty, consult the canonical C++ test implementation:
[HybridTestObjectCpp.cpp](https://github.com/mrousavy/nitro/blob/main/packages/react-native-nitro-test/cpp/HybridTestObjectCpp.cpp)

## Code Examples

### Type reference table

| TypeScript | C++ Type | Notes |
|-----------|----------|-------|
| `number` | `double` | Always `double`, never `float` |
| `string` | `const std::string&` (param) / `std::string` (return) | |
| `boolean` | `bool` | |
| `bigint` (signed) | `int64_t` | |
| `bigint` (unsigned) | `uint64_t` | |
| `T[]` | `std::vector<T>` | e.g. `std::vector<double>`, `std::vector<std::string>` |
| `Promise<T>` | `std::shared_ptr<Promise<T>>` | Use `Promise<T>::async(lambda)` |
| `Promise<void>` | `std::shared_ptr<Promise<void>>` | |
| `T \| undefined` | `std::optional<T>` | |
| `T \| U` | `std::variant<T, U>` | e.g. `std::variant<std::string, double>` |
| `(x: T) => void` | `std::function<void(T)>` | |
| `() => T` | `std::function<T()>` | |
| `ArrayBuffer` | `std::shared_ptr<ArrayBuffer>` | |
| `AnyMap` / `Record` | `std::shared_ptr<AnyMap>` | Nitro's generic map type |
| `Record<string, T>` | `std::unordered_map<std::string, T>` | For simple typed maps |
| `HybridObject` | `std::shared_ptr<HybridSpec>` | e.g. `std::shared_ptr<HybridMathSpec>` |
| `null` / `NullType` | `NullType` | Use `nitro::null` constant |
| `Date` | `std::chrono::system_clock::time_point` | |

### Throwing errors

```cpp
double HybridMath::divide(double a, double b) {
  if (b == 0) {
    throw std::runtime_error("Division by zero!");
  }
  return a / b;
}
```

### Callback parameter

```cpp
void HybridMath::compute(double input, std::function<void(double)> onResult) {
  double result = input * 2;
  onResult(result);
}
```

## Common Pitfalls

- **Wrong namespace** — The namespace must match `cxxNamespace` in `nitro.json` (e.g. `margelo::nitro::math`)
- **Forgetting `override`** — All virtual method implementations need `override`
- **Modifying generated spec** — Never edit `nitrogen/generated/` files
- **Using `float` instead of `double`** — Nitro uses `double` for all `number` types
- **Using `std::future<T>`** — Nitro does not use `std::future`; always use `std::shared_ptr<Promise<T>>` with `Promise<T>::async(...)`
- **Missing `TAG` member** — Required for `HybridObject(TAG)` constructor call

## Related Skills

- [native-nitrogen-codegen.md](native-nitrogen-codegen.md) — Must generate specs before implementing
- [spec-nitro-json.md](spec-nitro-json.md) — Configure `"c++"` in autolinking
- [native-implement-kotlin.md](native-implement-kotlin.md) — Android Kotlin alternative
- [native-implement-swift.md](native-implement-swift.md) — iOS Swift alternative
