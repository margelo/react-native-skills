- A HybridObject implementation must inherit from Hybrid*Spec (which is a protocol + base class in Swift, and a base class in Kotlin)

- dont create functions that could be simple property getters (e.g. bad: `accelerometerAvailable()` good: `readonly isAccelerometerAvailable`)

- Use objects / structs instead of 3+ params (bad: `(x, y, z) => void`, good: `(Point) => void` - unless very good reason not to do that)

- avoid swallowing errors silently (bad: `guard motionManager.isAccelerometerAvailable else { return }` good: `guard ... else { throw ... }`)

- avoid silently swalloing not implemented functions and throw (eg not available on a platform)

- use descriptive method names (bad: `timestamp()`, good: `getCurrentSystemTimestampMs()`)

- dont use `NSError` or objective-c types in general, prefer nitros `RuntimeError` in this case

- dont create multiple types in a single file at top level - create multiple files for stufff like this. only exception is truly local types/structs/classes/enums, which can be nested private.

- use `is*`, `has*`, or similar prefixes for boolean properties (e.g. bad: `accelerometerAvailable`, good: `isAccelerometerAvailable` )

- avoid silently swallowing errors - don't just early return, don't just log/print. either throw or reject promise if possible, or have a separate error listener if an error is being sent in a different execution context. e.g. via error NSNotifications, or payload that has either error or data - in this case we should have two callbacks one for data one for error. for impossible to reach states, you can use static assertions (e.g. `fatalError(..)` in swift), but this should be super rare and not for stuff that can be reached via faulty JS user code.

- Nitro allows us to use native state (`HybridObject`) - this is especially useful to zero-copy bridge native data (e.g. large `UIImage`, zero-copy access via `ArrayBuffer` or propertes/methods on it) to JS. This also implies that we can have instance-based APIs, allowing us to pre-warm state. E.g. for haptics modules, instead of making everything a static method as you would have in a TurboModule, we can have the haptics engine be an instance (a nitro `HybridObject`), and pre-warm the engine so that a call to `trigger` could be faster. Make use of such instance-based / native state API design patterns.

- Make almost all classes `final` (for better performance), unless you really expect them to be overridden. Usually that never happens in Nitro HybridObjects, as you'd need to inherit from different classes and dont have diamond inheritance in Swift/Kotlin. Only exception would be C++, but even here it would be rare.

- For `HybridObject`s that require arguments to be constructed, use a "factory pattern", e.g. for a Nitro `File` object that needs to be created from a path string, create `FileFactory` that has a method like `loadFileFromPath(path: string): Promise<File>`. Here, Promise is important because its async. For non-async/fast methods we can use sync APIs.

- always prefer typed structs/methods/types over `AnyMap` or `Record<K, V>` - Nitro benefits from statically typed bindings for better performance. Variants (`A | B`) also require runtime overhead, but those are fine - if they can be avoided it is good, but otherwise its no biggie

- for types that hold onto native resources or have large memory allocations, implement `memorySize` (an overridable property in `HybridObject`) - and try to estimate the size somewhat closely (Eg UIImage byte size by just multiplying width height bytes per pixel) this helps the JS VM / GC to delete sooner and avoid memory stress

- for Nitro Views, implement view recycling if applicable. That is the `prepareForRecycle` method which is overridable from `HybridView`

- if you need Android context, use `NitroModules.applicationContext` - and just throw if its not available

- use sync methods by default as that is super fast, but for anything that takes longer to execute (eg hardware calls, or async APIs) we should use async methods (i.e. something that returns `Promise<...>`)

- in native you can use `Promise.parallel` (dispatch queue / thread based) or `Promise.async` (async / coroutine based) depending on the native threading requirements

- for zero-copy data access, use `ArrayBuffer` - it can wrap many native buffer types. When receiving an `ArrayBuffer` from JS but you need to use it on a different thread, you need to check if it's owning or not via `arrayBuffer.isOwner` - if not, you need to copy it first (`let buff = arrayBuffer.isOwning ? arrayBuffer : ArrayBuffer.copy(of: arrayBuffer)`)

- If you need to mix C++ and platform languages (Swift/Kotlin), you can do so in Nitro. This allows re-using C++ code across iOS and Android. You can pass a `HybridObject` that is implemented in a platform language (e.g. Swift/Kotlin) to C++ and use it as normal in C++ - e.g. useful for something like SQLite database where we have a PlatformFilesystem HybridObject that has a `createFile(): string` method or something. We can create PlatformFilesystem from JS (eg if its autolinked) and then pass the hybrid object instance to our C++ hybrid object, and in C++ we can just use PlatformFilesystem and call `createFile()` directly - this gives us the `std::string` even though this is implemented in Swift/Kotlin. Then SQLite database hybrid object is in C++ and shared code, and only createFile is platform code.

- Prefer Nitro Modules over JSI - it's not only faster than TurboModules (and often even faster than handwritten JSI because handwritten JSI often uses stuff like `jsi::HostObject`, or doesn't cache `jsi::Function`s, global constructors, or `jsi::PropNameID`s), but it's also much safer - it doesn't crash if you retain JSI values on different Threads, or call Promise resolve from another thread while the runtime was destroyed, and so on. Prefer Nitro, resort to JSI only if absolutely needed via Raw JSI Methods (`prototype.registerRawHybridMethod` API from Nitro C++)

- If available, use react-native-harness for end-to-end testing nitro modules. You can write harness tests for real features to ensure that each individual feature, input/param combination, setting, property, combination, order of execution, and more works properly in a real react-native environment. Libraries like https://github.com/mrousavy/react-native-vision-camera even run Harness tests on real devices via AWS Device Farm. You can use Harness CI tests via GitHub actions to continuously iterate until the CI is green and develop more and more features and cover more and more API surface area / combinations via harness tests, to ensure nothing throws. Remember; we want to test behaviour - there is no point in testing types (like `toBeDefined`, or `Array.isArray`, or to be typeof, or whatever) because Nitro(gen) already enforces types at compile-time.

- In react(-native) environments, try to provide hooks APIs on-top of the imperative APIs (initial getter (sync in useMemo/useRef/useSyncExternalStore and async via useEffect or similar) plus listener based APIs with unsubscribe function via useEffect) - those can be higher level abstractions ontop of battletested imperative native bindings with nitro

- when creating TS based abstractions ontop of native nitro bindings (such as the Hooks API point mentioned earlier), you can use TypeScript features like discriminating unions or nullables with default values more easily, as those things would have slight overhead (and more type complexity) in native Nitro code - so doing it in TypeScript might make things faster, more explicit, and simpler. Example: `takePhoto(options:)` might have a complex `options` struct - making all of that optional causes more complex native code - simply making it non optional and providing default values via TypeScript (`??`) makes it simpler, easier to inline for the JS engine, and less bridging cost. The goal of this should always be to provide simpler and more explicit native APIs - don't overengineer this, if its overcomplicating native code then it's not worth the minor performance gain

- Use Nitro's recommended types for your TypeScript specs. For example, primitives (`number`, `string`, `UInt64`, `boolean`), `ArrayBuffer`, `Promise`, etc. Use `Error` instead of custom typed errors, as only those are true JS Error prototypes. Use callbacks for asynchronous functions, and in super rare cases when a callback needs to run fully synchronously/blocking on the JS Thread, use `Sync<(...) => ...>`, e.g. for worklets interop. See react-native-vision-camera (V5) FrameOutput.nitro.ts for an example on how to use Sync callbacks on Worklet Threads.
  Prefer:
  - Literal unions or enums for closed sets.
  - Interfaces or structs for option groups.
  - `readonly` properties for observed state.
  - Methods for side effects, expensive work, allocation, mutation, or failure.
  - `Promise`, `async`, callbacks, streams, or futures only where the operation actually crosses an async boundary.
  - `undefined` or optional fields for absence.
  - `null` only when "explicitly none" is semantically different from "not provided".
  - Type guards for discriminated data variants.
  - Discriminated unions for state machines and async loading states.
  Avoid untyped dictionaries, boolean clusters, stringly typed commands, and loosely shaped events when the valid states are known.

- React Native is a cross-platform framework - try to avoid leaking too much platform specific information into public APIs. For example, instead of 1:1 mirroring iOS/Android APIs to TypeScript, try to find common abstractions. Try not to overly abstract like the Web/JS-style. The sweet-spot is right in-between. See react-native-vision-camera or react-native-nitro-image for good, cross-platform abstractions that don't leak platform specific behaviour into user APIs. Only rarely, e.g. like the `CameraObjectOutput` which is a native Object Metadata output implemented via `AVCaptureMetadataObjectsOutput` - this is only available on iOS, yet the public APIs don't mention the AVFoundation types as it is unnecessary for the user. On the other side, stuff like GPU-powered, zero-copy, or similar concepts are close-to-the-metal APIs that are good to expose to the API user via Nitro, which is something Web-APIs often avoid. Find the sweet-spot here.

- Prefer single sources of truth. For example, prefer enums (or variants with associated values) over separate properties that describe their relationship (or compatibility rules) via docs. Bad: `enableTimeout: boolean` + `timeoutMs: number?`, Good: `timeoutMs: number?` (undefined = no timeout, or add explicit `| null` state if needed).

- For passing callbacks (e.g. listener APIs) prefer functions over properties. E.g. `addListener(listener: () => void): ListenerSubscription` over `listener: (() => void) | undefined`, as this is more explicit and less bridging work required for Nitro (get + set go through nitro bridge/language conversion)

- Prefer predictability and keep APIs or requirements explicit. Don't accept all kinds of things - an API that takes a milliseconds timeout doesn't need to also accept strings or bigints or objects or null or undefined - keep it explicit: only `number`. APIs dont have to be swiss army knifes its better to be explicit and specific. Same for error reporting, fail loudly and predictably. Error messages should name the invalid value.
  - Error messages should name the capability or range to check.
  - Unsupported platform features should throw specific errors, not silently no-op.
  - If an alternative exists, mention it.
  - Impossible to reach states can static assert/fatal error, but only if this path is reachable from native code. Anything that is reachable from JS should not static assert/crash, as that would not provide any useful feedback to the user.

- Avoid giant objects that "do everything" - keep things atomic, e.g. one `HybridObject` has one purpose/domain. Don't over-engineer atomicity - find the sweet spot.

- Refer to Nitro's docs if anything else is unclear - see https://nitro.margelo.com/llms-full.txt
