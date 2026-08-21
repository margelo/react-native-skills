---
name: react-native-vision-camera-realtime
description: Design and review production-grade low-latency VisionCamera v5 pipelines. Use for real-time GPU, ML, CV, Skia or WebGPU overlays, Nitro frame plugins, zero-copy interop, frame budgets, and latency profiling. Use the general react-native-vision-camera skill for setup, capture, controls, basic frame outputs, or v4 migration.
---

# Real-time VisionCamera pipelines

This is the specialized companion to `react-native-vision-camera`. Optimize the complete path from Camera buffer to final result, not an isolated stage. Before relying on exact APIs, check installed versions against current [VisionCamera docs](https://visioncamera.margelo.com/llms.txt) and the consumer's official docs or source.

## Choose by final consumer

| Final consumer | Preferred path |
|---|---|
| Frame-coupled rendering, effects, or overlays | Keep processing and drawing on one GPU timeline with `<SkiaCamera />` or WebGPU |
| WGSL compute or GPU inference | `Frame.getNativeBuffer()` to a WebGPU video frame to `device.importExternalTexture(...)` |
| Native plugin that depends on VisionCamera | A long-lived Nitro HybridObject whose hot method accepts a typed `Frame` |
| Native library without a VisionCamera dependency | The untyped `NativeBuffer` pointer and explicit release contract |
| State-only ML or scanning | Benchmark the platform runtime across ANE or NPU, GPU, and CPU backends; return compact state |
| CPU-only consumer | Use the smallest useful resolution and format with a bounded, reusable CPU buffer path |

Load [references/interop.md](references/interop.md) only when implementing or reviewing Nitro, NativeBuffer, WebGPU, Skia, Resizer, or `ArrayBuffer` interop.

## Hot-path invariants

1. Keep orientation and mirroring as metadata. Set `enablePhysicalBufferRotation: false`, then pass `frame.orientation` and `frame.isMirrored` to the consumer or apply them in the same GPU transform that scales, crops, or renders. Never rotate the Camera buffer physically.
2. Stay in one execution and memory domain. In a GPU pipeline, import once, keep preprocessing, inference, postprocessing, and rendering on the GPU, and read back only a compact result when required.
3. Prefer `pixelFormat: 'native'` for a verified GPU-only path. Check `frame.pixelFormat` and `frame.hasNativeBuffer` because the resolved native format may be YUV, RGB, RAW, or private.
4. Do not use `getPixelBuffer()`, `getPlanes()`, plane pixel buffers, mapped GPU buffers, or typed pixel views in the normal GPU path. CPU visibility can force synchronization or download.
5. Create and warm pipelines, shaders, samplers, model sessions, resizers, large buffers, and native processors once. Reuse them for the component or session lifetime; never allocate them per frame.
6. Draw frame-coupled overlays from the same `Frame` with Skia or WebGPU. Do not route per-frame geometry through React state, ordinary views, or Reanimated shared values.
7. Release every `Frame`, `NativeBuffer`, wrapper, texture, and pooled slot exactly once on every path. Release wrappers in reverse ownership order and dispose the `Frame` last.

## Prefer same-frame processing

Keep detection, tracking, decisions, and drawing synchronous with the matching frame when they must align visually. At 60 FPS the hard interval is 16.67 ms; at 30 FPS it is 33.33 ms. Target under roughly 16 ms and 33 ms to leave scheduling margin.

"Synchronous" means same-frame dataflow, not blocking the CPU until the GPU finishes. Encode dependent GPU stages in one command graph when possible. Do not add per-frame `queue.onSubmittedWorkDone()`, buffer mapping, readback, or another CPU or GPU fence.

Before making work asynchronous, remove copies and readbacks, reduce input resolution or FPS, fuse passes, optimize model tensors, and reuse warmed state. Use async only when the optimized work still cannot fit the frame interval, often around 50 ms or more, and the product accepts stale results. For frame-coupled visuals, prefer simplifying the work over visible lag.

The async delivery patterns are peers:

- native Nitro work with a retained completion callback
- native Nitro work that stores completed state behind a synchronous latest-state getter
- a synchronous native method scheduled with VisionCamera's `useAsyncRunner()`

Every async design must bound in-flight work. Use one active task or a small fixed pool, reject or replace stale pending input, and never build an unbounded FIFO queue. `dropFramesWhileBusy` is an overload guard, not the architecture. With `useAsyncRunner()`, dispose an accepted `Frame` inside the task and a rejected `Frame` immediately.

## Choose ML compute end to end

If inference feeds a same-frame Skia or WebGPU render, prefer keeping the entire path on the GPU. Crossing to an ANE, NPU, or CPU and returning geometry to the renderer is worthwhile only when end-to-end profiling proves it is faster while preserving the frame budget.

For state-only scanning, benchmark the platform runtime's available compute units. An ANE or NPU can avoid GPU contention and accelerate supported models; a CPU can win for tiny models when accelerator dispatch and transfer cost dominates. Measure input conversion, synchronization, inference, and result delivery, not inference alone. Normal React state or navigation is fine after a scan that has no frame-coupled overlay.

## Development and production checks

When all native dependencies support it, use a resizable iPad-shaped Mac Catalyst or iPad-on-Mac build as a rapid iteration harness. A desktop agent can relaunch, resize, and screenshot it while using a built-in Mac camera or external UVC camera via `useCameraDevice('external')`. Fall back to a phone when the Mac target or required plugin is unavailable.

The Mac loop is for functional iteration, not performance prediction. Validate release builds on every production device class and representative GPUs. Test long enough to expose thermal throttling and pool leaks. Track:

- camera timestamp to matching result or presentation latency at median, p95, and p99
- dropped frames and maximum in-flight frames
- CPU and GPU time, readbacks, maps, and synchronization points
- allocations per frame, steady-state memory, sustained FPS, temperature, and power

Sample GPU timings asynchronously and sparsely enough that instrumentation does not become a synchronization point.

## Authoritative references

- VisionCamera: [docs index](https://visioncamera.margelo.com/llms.txt), [performance](https://visioncamera.margelo.com/docs/performance), [async processing](https://visioncamera.margelo.com/docs/async-frame-processing), [external cameras](https://visioncamera.margelo.com/docs/devices)
- Rendering and compute: [VisionCamera Skia](https://visioncamera.margelo.com/docs/skia-frame-processors), [React Native WebGPU integration](https://github.com/wcandillon/react-native-webgpu/blob/main/apps/docs/content/docs/integrations/vision-camera.mdx)
- ML compute: [Apple Core ML compute units](https://developer.apple.com/documentation/coreml/mlcomputeunits), [LiteRT NPU delegates](https://ai.google.dev/edge/litert/android/npu)
