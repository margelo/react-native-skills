---
name: react-native-vision-camera-realtime
description: Design, implement, and review low-latency real-time processing pipelines for React Native VisionCamera v5. Use for frame processors, GPU or ML or CV pipelines, Skia or WebGPU interop, Nitro plugins, orientation, buffer ownership, frame budgets, backpressure, and production performance. Do not use for ordinary camera setup, permissions, photo or video capture, or v4 migration.
---

# Low-latency VisionCamera pipelines

Optimize the whole path from Camera buffer to final consumer. A fast individual stage does not make a fast pipeline if another stage rotates pixels, converts formats, maps GPU memory to the CPU, copies buffers, blocks for completion, or queues stale frames.

Before using exact APIs, check the installed package versions against the current [VisionCamera `llms.txt`](https://visioncamera.margelo.com/llms.txt), [VisionCamera source](https://github.com/mrousavy/react-native-vision-camera), and the relevant consumer's docs or source. VisionCamera v5, React Native WebGPU, and their interop APIs evolve quickly.

## Choose the pipeline by its final consumer

| Consumer | Preferred path |
|---|---|
| Skia rendering, effects, or overlays | `react-native-vision-camera-skia` for the fastest prototype, or `Frame.getNativeBuffer()` with `Skia.Image.MakeImageFromNativeBuffer(...)` for a custom renderer |
| WGSL rendering, CV kernels, or GPU inference | `Frame.getNativeBuffer()` to `RNWebGPU.createVideoFrameFromNativeBuffer(...)` to `device.importExternalTexture(...)` |
| A native library that intentionally depends on VisionCamera | A Nitro `HybridObject` method that accepts `Frame` directly, then unwraps the typed native frame |
| A third-party native library that must not depend on VisionCamera | The untyped `NativeBuffer` contract, with explicit retain and release ownership |
| A CPU-only model or algorithm | A deliberately bounded fallback using the lowest useful resolution and a consumer-compatible pixel format |

Do not choose an `ArrayBuffer` path merely because it is easy to prototype. Choose it only when the actual consumer needs CPU-visible bytes.

## Non-negotiable hot-path rules

1. Keep orientation and mirroring as metadata. Set `enablePhysicalBufferRotation: false`, then pass `frame.orientation` and `frame.isMirrored` to the consumer or apply the inverse transform in a GPU render or compute pass. Do not physically rotate or mirror the Camera buffer.
2. Stay in one execution and memory domain for as long as possible. For a GPU pipeline, import the Camera buffer once, run preprocessing, inference, postprocessing, and rendering on the GPU, and read back only the small final result that the app truly needs.
3. Prefer `pixelFormat: 'native'` for a verified GPU-only path. Check `frame.pixelFormat` and `frame.hasNativeBuffer` at runtime because the negotiated native format can be YUV, RGB, RAW, or private. Constrain or fall back when the consumer cannot import the resolved format.
4. Do not call `frame.getPixelBuffer()`, `frame.getPlanes()`, plane `getPixelBuffer()` methods, or create typed pixel views in the normal GPU hot path. These APIs do not necessarily copy immediately, but they make pixels CPU-accessible and can lazily trigger a GPU-to-CPU download or synchronization.
5. Never allocate pipelines, shader modules, samplers, large buffers, model sessions, resizers, or native processors per frame. Treat them as long-lived state. With Nitro, create a processor HybridObject once for the component or session, often through an asynchronous factory, and let it own and reuse the warmed resources for as long as the HybridObject is alive.
6. Keep frame-dependent processing and visual feedback on the same `Frame` whenever possible. Detection, tracking, and drawing that belong together should remain one synchronous frame pipeline and fit within the frame interval. Do not make a pipeline asynchronous merely to hide an avoidable slow path.
7. Release every retained resource on every path. A leaked `Frame`, `NativeBuffer`, external texture, video-frame wrapper, resized frame, or pooled slot eventually stalls the Camera or grows memory.

Start a GPU frame output explicitly:

```ts
const frameOutput = useFrameOutput({
  pixelFormat: 'native',
  enablePhysicalBufferRotation: false,
  targetResolution: modelOrRendererResolution,
  onFrame(frame) {
    'worklet'
    try {
      processOnGpu(frame)
    } finally {
      frame.dispose()
    }
  },
})
```

`enablePhysicalBufferRotation` already defaults to `false`, but keep it explicit in performance-sensitive examples so later edits do not silently add a physical conversion.

## Orientation and mirroring

`frame.orientation` describes how the pixels are rotated relative to the output's requested orientation. `frame.isMirrored` describes the remaining mirror difference. Together they are the presentation recipe.

- If a native ML API accepts orientation and mirror flags, pass the metadata directly.
- In Skia, counter-rotate and counter-mirror through the canvas, image matrix, or shader sampling transform. Do not create a rotated image buffer.
- React Native WebGPU extends `importExternalTexture(...)` with `rotation` and `mirrored`. Follow its current VisionCamera mapping rather than inventing texture-coordinate logic. Current sources map `up`, `right`, `down`, and `left` to `0`, `90`, `180`, and `270` respectively, and pass `frame.isMirrored` as `mirrored`.
- The VisionCamera Resizer already counter-rotates and counter-mirrors. Do not apply the metadata twice after resizing.

If a generic renderer has no metadata options, apply the inverse orientation in the same GPU pass that scales, crops, converts color, or renders. Fuse transforms instead of adding another full-frame pass when practical.

## Prefer typed `Frame` interop for VisionCamera plugins

When a native processor intentionally depends on VisionCamera, model that dependency in its Nitro spec:

```ts
import type { HybridObject } from 'react-native-nitro-modules'
import type { Frame } from 'react-native-vision-camera'

export interface Detector
  extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  process(frame: Frame): void
}

export interface DetectorFactory
  extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  createDetector(modelPath: string): Promise<Detector>
}
```

Create the factory as the default-constructible autolinked root, then call `createDetector(...)` once during component or session initialization. The factory may compile and warm the native pipeline asynchronously; its Promise should resolve with a ready `Detector`. Retain that `Detector` for the component or session lifetime and call only its hot `process(frame)` method from `onFrame`.

The native `Detector` implementation owns the compiled state as members, such as an `MTLComputePipelineState`, model session, GPU context, scratch textures, command resources, and pools. Their lifetime follows the HybridObject instead of the individual `process(...)` call. Release them when the HybridObject dies, and report their retained size through `memorySize` when they own significant memory.

This preserves type safety and lets native implementations unwrap the platform frame without exposing raw pointers to JS:

- Swift: cast `HybridFrameSpec` to VisionCamera's `NativeFrame`, then access its `CMSampleBuffer`.
- Kotlin: cast `HybridFrameSpec` to `NativeFrame`, then access its `ImageProxy`.
- C++: use the generated `HybridFrameSpec` API and `getNativeBuffer()` for platform buffer interop. A platform frame cannot be reliably downcast in shared cross-platform C++.

Keep hot state such as model sessions, GPU contexts, scratch textures, command resources, and buffer pools on the long-lived HybridObject. Return small scalar result structs when cheap. Keep large or lazily accessed native results behind HybridObjects instead of eagerly materializing nested JS objects or byte arrays.

## Use `NativeBuffer` for dependency-free interop

`Frame.getNativeBuffer()` is the shared, untyped escape hatch for libraries that should not take a hard VisionCamera dependency. Check `frame.hasNativeBuffer` first. The returned object contains:

- `pointer: bigint`, representing `CVPixelBufferRef` on iOS or `AHardwareBuffer*` on Android
- a `release()` function for the extra retain acquired by `getNativeBuffer()`

Release the consumer wrapper before releasing the `NativeBuffer`, and release the `NativeBuffer` before disposing the `Frame`. Use `try` and `finally` so failure paths follow the same lifetime order.

### React Native WebGPU

The current zero-copy import path is:

```ts
const webGpuRotations = {
  up: 0,
  right: 90,
  down: 180,
  left: 270,
} as const

function submitWebGpuFrame(frame: Frame) {
  'worklet'
  try {
    if (!frame.hasNativeBuffer) return
    const nativeBuffer = frame.getNativeBuffer()
    try {
      const videoFrame = RNWebGPU.createVideoFrameFromNativeBuffer(nativeBuffer.pointer)
      try {
        const externalTexture = device.importExternalTexture({
          source: videoFrame,
          rotation: webGpuRotations[frame.orientation],
          mirrored: frame.isMirrored,
        })
        try {
          const commandEncoder = encodeGpuWork(externalTexture)
          device.queue.submit([commandEncoder.finish()])
          // If rendering to a Canvas, call context.present() here.
        } finally {
          externalTexture.destroy()
        }
      } finally {
        videoFrame.release()
      }
    } finally {
      nativeBuffer.release()
    }
  } finally {
    frame.dispose()
  }
}
```

This helper owns the passed `Frame` and disposes it exactly once. Do not add another disposal wrapper around the same call.

Import an external texture for each frame. It is short-lived and expires after submitted work, so do not cache it. Cache the `GPUDevice`, pipelines, layouts, shaders, samplers, static bind groups, and reusable uniform, storage, and output buffers.

An external texture can be sampled from WGSL render or compute work. For multiple CV or ML stages, sample or normalize once into shared GPU textures or buffers, run the stages on the same device, and submit a coherent command graph. Do not map intermediate `GPUBuffer`s or wait with `queue.onSubmittedWorkDone()` every frame. If results must reach JS, asynchronously read back only the compact result and use multiple readback slots so the GPU does not wait for the CPU.

Follow React Native WebGPU's current feature requirements and platform-specific YUV handling. Do not assume that iOS and Android expose identical sampled channel values.

### React Native Skia

For a quick shader or overlay prototype, prefer `<SkiaCamera />` and its provided `frameTexture` and canvas. If no custom Skia drawing is needed, use the regular `<Camera />` because attaching a Skia frame output adds work.

For a custom Skia renderer, call `Skia.Image.MakeImageFromNativeBuffer(nativeBuffer.pointer)`, draw with a GPU matrix derived from `orientation` and `isMirrored`, then dispose the `SkImage`, release the `NativeBuffer`, and dispose the `Frame`. Reuse the Skia surface, paints, runtime effects, and other drawing resources.

## CPU access and the Resizer fallback

VisionCamera's `getPixelBuffer()` is zero-copy at the JS/native binding boundary, but source inspection confirms it may lazily perform a GPU-to-CPU download. `getPlanes()` exposes the same CPU pixel domain one plane at a time. Treat both as fallback APIs, not GPU interop APIs.

`react-native-vision-camera-resizer` performs resize, format conversion, orientation, and mirroring on Metal or Vulkan and returns a pooled `GPUFrame`. It is a strong choice when the next consumer requires a small CPU-visible tensor, and often the fastest way to prototype an existing ArrayBuffer-based model. It is not proof of an end-to-end GPU pipeline: calling `GPUFrame.getPixelBuffer()` for a CPU consumer introduces CPU visibility and possible synchronization.

When latency is the priority and the model stack supports WebGPU, import the Camera frame as an external texture and keep preprocessing and inference in GPU compute passes. For multiple models, share the imported frame and common preprocessing outputs instead of resizing, converting, downloading, and uploading independently for each model.

If CPU access is unavoidable:

- Negotiate the smallest useful `targetResolution` and no more FPS than the consumer can sustain.
- Prefer YUV when the CPU consumer supports it. Do not force RGB in the Camera pipeline unless the measured consumer path is faster overall with RGB.
- Run CPU work off the UI thread with bounded backpressure.
- Keep CPU buffers native-owned and reusable.
- Measure the complete path, including conversion, synchronization, model execution, and result delivery.

## Reuse Nitro `ArrayBuffer`s safely

Do not allocate and return a new large `ArrayBuffer` from a frame processor plugin for every frame. If CPU output is unavoidable, let the long-lived processor HybridObject hold an owning `ArrayBuffer` member, allocated once with Nitro's `ArrayBuffer.allocate(...)`, wrapped around existing owned memory without a copy, or created from JS with `NitroModules.createNativeArrayBuffer(size)`. Update and reuse that buffer instead of reallocating it in `process(frame)`.

An `ArrayBuffer` is CPU-visible memory, so reuse only avoids allocation and transfer overhead around a CPU path. It does not preserve an end-to-end GPU pipeline. Prefer keeping output in a GPU texture or buffer when the next stage can consume it there, and expose or read an `ArrayBuffer` only for a consumer that actually needs CPU bytes.

Nitro `ArrayBuffer`s are not thread-safe. A single reusable buffer is valid only when there is exactly one in-flight writer and the consumer's access is synchronously scoped before the next write. If processing or reading can overlap, use a small fixed ring pool with explicit acquire and release ownership. Never overwrite a slot still visible to JS or another native thread.

A normal JS-created `ArrayBuffer` is non-owning from native's perspective and is safe to access only during the synchronous Nitro call. Do not retain it or use it after a thread hop. Prefer a native-owned reusable buffer over copying a non-owning buffer on every frame.

## Prefer same-frame processing

Keep the full frame-dependent decision and submission synchronous by default. Consume one `Frame`, run or encode its dependent detection, tracking, and rendering stages, and associate any visual feedback with that exact frame before `onFrame(...)` returns. Hand landmarks, face boxes, masks, and overlays otherwise lag behind motion when they are computed from an older frame and drawn over a newer one.

"Synchronous" describes the same-frame dataflow, not a CPU wait for the GPU. Encode ordered GPU passes and submit them as one coherent command graph when possible. Do not call `queue.onSubmittedWorkDone()`, map a result buffer, or add another CPU or GPU completion fence per frame. For WebGPU, release the `Frame` after the commands that consume its external texture have been submitted in the documented ownership order.

At 60 FPS the hard frame interval is 16.67 ms; at 30 FPS it is 33.33 ms. Target under roughly 16 ms and 33 ms respectively to leave scheduling margin. Before introducing asynchronous delivery, verify that the pipeline already avoids copies and readbacks, stays on the GPU, reuses warmed state, fuses compatible passes, uses an appropriate input resolution and FPS, and has optimized model tensors and execution.

Use asynchronous processing only when profiling shows that the work still cannot fit the frame interval and the product can tolerate results from an older frame. A pipeline that still takes roughly 50 ms or more after those optimizations is a reasonable async candidate, but 50 ms is an example rather than a universal cutoff. For frame-coupled visual feedback, prefer reducing or replacing the expensive work over accepting visible lag.

The following async delivery patterns are equally valid. Choose according to API shape and ownership needs:

- Let a native Nitro processor start bounded asynchronous work and invoke a retained callback with each completed result.
- Let a native Nitro processor store the latest completed state and expose a synchronous getter. The frame processor polls that state without a JS callback.
- Keep the native processor method synchronous and schedule it on VisionCamera's dedicated runtime with `useAsyncRunner()`.

Once async is justified, bound the number of in-flight frames. Do not create an unbounded FIFO queue. Prefer one active task or a small fixed pool, reject or replace stale pending input, and publish only completed state. This is overload containment for an intentionally asynchronous pipeline, not a reason to make a healthy same-frame pipeline asynchronous.

`dropFramesWhileBusy` is likewise an overload guard, not the architecture. A healthy synchronous pipeline should finish before the next frame, so the guard should not activate during steady state. With `useAsyncRunner()`, dispose the `Frame` inside an accepted task and immediately when `runAsync(...)` rejects it because the runner is busy.

Configure only the Camera outputs the feature actually needs. Measure camera timestamp to matching result or presentation latency as well as stage duration, because average throughput can look healthy while asynchronous visual feedback remains perceptibly behind.

## Production verification

Validate release builds on physical iOS and Android devices across representative GPU vendors. Test cold start and sustained runs long enough to expose thermal throttling and pool leaks.

Track at least:

- camera timestamp to result or presentation latency at median, p95, and p99
- dropped frames and maximum in-flight frames
- CPU time, GPU time, and any readback or map operations
- allocations per frame and steady-state memory
- sustained FPS, device temperature, and power behavior

Instrumentation must not become a synchronization point. Read GPU timestamps or compact diagnostics asynchronously and at a lower sampling rate if per-frame measurement changes the pipeline.

## Authoritative references

- VisionCamera docs index: https://visioncamera.margelo.com/llms.txt
- VisionCamera performance: https://visioncamera.margelo.com/docs/performance
- VisionCamera orientation: https://visioncamera.margelo.com/docs/orientation
- VisionCamera `Frame`: https://visioncamera.margelo.com/docs/a-frame
- VisionCamera `NativeBuffer`: https://visioncamera.margelo.com/docs/a-frames-nativebuffer
- VisionCamera native plugins: https://visioncamera.margelo.com/docs/native-frame-processor-plugins
- VisionCamera Skia integration: https://visioncamera.margelo.com/docs/skia-frame-processors
- VisionCamera Resizer: https://visioncamera.margelo.com/docs/resizer
- VisionCamera async frame processing: https://visioncamera.margelo.com/docs/async-frame-processing
- React Native WebGPU VisionCamera integration: https://github.com/wcandillon/react-native-webgpu/blob/main/apps/docs/content/docs/integrations/vision-camera.mdx
- React Native WebGPU native extensions: https://github.com/wcandillon/react-native-webgpu/blob/main/apps/docs/content/api/gpu-device-extensions.mdx
- React Native Skia source: https://github.com/Shopify/react-native-skia
- Nitro `ArrayBuffer` ownership and threading: https://nitro.margelo.com/docs/types/array-buffers
- Nitro callbacks: https://nitro.margelo.com/docs/types/callbacks
