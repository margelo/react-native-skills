# GPU and native interop

Read this reference only for implementation work involving Nitro, `NativeBuffer`, WebGPU, Skia, the Resizer, or CPU-visible buffers. Confirm exact APIs against the installed package versions and current official source.

## Orientation and mirroring

- Pass metadata directly when a native ML API accepts it.
- For Skia, counter-rotate and counter-mirror through the canvas, image matrix, or shader sampling transform.
- React Native WebGPU accepts `rotation` and `mirrored` in `importExternalTexture(...)`. Current VisionCamera mapping is `up: 0`, `right: 90`, `down: 180`, and `left: 270`, with `frame.isMirrored` passed as `mirrored`.
- The VisionCamera Resizer already counter-applies orientation and mirroring. Do not apply the metadata twice.

## Typed `Frame` Nitro plugins

Use a typed `Frame` when the native processor intentionally depends on VisionCamera:

```ts
import type { HybridObject } from 'react-native-nitro-modules'
import type { Frame } from 'react-native-vision-camera'

export interface DetectedResult {
  x: number
  y: number
  confidence: number
}

export interface Detector
  extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  process(frame: Frame): DetectedResult[]
}

export interface DetectorFactory
  extends HybridObject<{ ios: 'swift'; android: 'kotlin' }> {
  createDetector(modelPath: string): Promise<Detector>
}
```

Keep the factory default-constructible. Call `createDetector(...)` once, await a compiled and warmed `Detector`, and retain it for the component or session lifetime. The detector owns model sessions, GPU contexts, pipeline state, scratch resources, and pools as members. Implement `memorySize` when it retains substantial native memory.

Return small scalar structs synchronously when cheap. Keep large, lazy, binary, or native-backed results behind HybridObjects instead of eagerly converting them to JS values.

Native unwrapping is platform-specific:

- Swift: cast `HybridFrameSpec` to VisionCamera's `NativeFrame`, then access its `CMSampleBuffer`.
- Kotlin: cast `HybridFrameSpec` to `NativeFrame`, then access its `ImageProxy`.
- C++: use the generated `HybridFrameSpec` API and `getNativeBuffer()`. Do not assume a shared C++ implementation can downcast the platform frame.

## `NativeBuffer` ownership

Use `Frame.getNativeBuffer()` for dependency-free interop after checking `frame.hasNativeBuffer`. Its `pointer` is a retained `CVPixelBufferRef` on iOS or `AHardwareBuffer*` on Android. The consumer must release the extra retain.

Acquire in this order:

1. `Frame`
2. `NativeBuffer`
3. consumer wrapper, such as a WebGPU video frame or Skia image
4. imported texture or other temporary view

Release in reverse order after submitting the work that consumes the resource. Use nested `try` and `finally`; dispose the `Frame` last. Do not retain any layer longer than required.

## WebGPU

The zero-copy path is `Frame.getNativeBuffer()` to `RNWebGPU.createVideoFrameFromNativeBuffer(...)` to `device.importExternalTexture(...)`:

```ts
const rotation = { up: 0, right: 90, down: 180, left: 270 } as const

function submit(frame: Frame) {
  'worklet'
  try {
    if (!frame.hasNativeBuffer) return
    const buffer = frame.getNativeBuffer()
    try {
      const source = RNWebGPU.createVideoFrameFromNativeBuffer(buffer.pointer)
      try {
        const texture = device.importExternalTexture({
          source,
          rotation: rotation[frame.orientation],
          mirrored: frame.isMirrored,
        })
        try {
          device.queue.submit([encodeGpuWork(texture).finish()])
        } finally {
          texture.destroy()
        }
      } finally {
        source.release()
      }
    } finally {
      buffer.release()
    }
  } finally {
    frame.dispose()
  }
}
```

Import the external texture per frame because it expires after submitted work. Cache the device, pipelines, layouts, shaders, samplers, static bind groups, and reusable buffers. Share imported inputs and preprocessing outputs across multiple models. Do not map intermediate buffers or wait for queue completion per frame. Read back only compact results, asynchronously, through a small ring of reusable slots.

Verify current platform-specific YUV behavior and feature requirements. Do not assume identical sampled channels on iOS and Android.

## Skia

Use `<SkiaCamera />` and its `frameTexture` and canvas for the fastest frame-coupled prototype. Use a regular `<Camera />` when no custom drawing is needed because a Skia frame output adds work.

For a custom renderer, create a `SkImage` with `Skia.Image.MakeImageFromNativeBuffer(...)`, draw with the orientation and mirror matrix, then dispose the image, release the `NativeBuffer`, and dispose the `Frame`. Reuse surfaces, paints, and runtime effects.

## CPU and `ArrayBuffer` fallbacks

`getPixelBuffer()`, `getPlanes()`, and plane pixel buffers expose the CPU pixel domain and may trigger a GPU download or synchronization. Use them only for a consumer that requires CPU-visible pixels.

The VisionCamera Resizer performs resize, conversion, orientation, and mirroring on Metal or Vulkan, but calling `GPUFrame.getPixelBuffer()` still ends in CPU-visible output. It is appropriate for a small CPU tensor, not proof of an end-to-end GPU pipeline.

If CPU access is unavoidable:

- negotiate the smallest useful resolution and no more FPS than the consumer sustains
- prefer YUV when supported; force RGB only when the measured consumer path is faster overall
- keep CPU work off the UI thread and reuse native-owned memory
- include conversion, synchronization, execution, and delivery in measurements

Do not allocate a large returned `ArrayBuffer` per frame. Let the long-lived processor HybridObject own an `ArrayBuffer` allocated once with Nitro, or a small fixed ring when access can overlap. Nitro `ArrayBuffer`s are not thread-safe, so one reusable buffer requires exactly one in-flight writer and synchronously scoped readers. A normal JS-created `ArrayBuffer` is non-owning from native's perspective and must not survive the synchronous Nitro call.

## Sources

- VisionCamera: [orientation](https://visioncamera.margelo.com/docs/orientation), [`Frame`](https://visioncamera.margelo.com/docs/a-frame), [`NativeBuffer`](https://visioncamera.margelo.com/docs/a-frames-nativebuffer), [native plugins](https://visioncamera.margelo.com/docs/native-frame-processor-plugins), [Resizer](https://visioncamera.margelo.com/docs/resizer)
- React Native WebGPU: [VisionCamera integration](https://github.com/wcandillon/react-native-webgpu/blob/main/apps/docs/content/docs/integrations/vision-camera.mdx), [native extensions](https://github.com/wcandillon/react-native-webgpu/blob/main/apps/docs/content/api/gpu-device-extensions.mdx)
- Nitro: [`ArrayBuffer` ownership and threading](https://nitro.margelo.com/docs/types/array-buffers), [callbacks](https://nitro.margelo.com/docs/types/callbacks)
