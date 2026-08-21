---
name: react-native-vision-camera
description: Best-practices guide for React Native VisionCamera v5 setup, migration, capture, controls, outputs, and basic frame processing. Use the separate react-native-vision-camera-realtime skill for production low-latency GPU, ML, CV, Skia or WebGPU pipelines and frame-coupled overlays.
---

# react-native-vision-camera (v5)

VisionCamera v5 is the maintained and latest version of `react-native-vision-camera`. It is a full Nitro Modules rewrite with a new **Constraints API**, **Output-based architecture**, **in-memory `Photo`**, and a hard break from the v4 format/prop model. Almost every v4 surface is gone or renamed — treat v5 as a new API, not an incremental upgrade.

This skill is a router. Read this file first, then load the reference that matches the task. Every reference is self-contained — do not load more than you need.

## When to load which reference

- **Production low-latency GPU, ML, CV, or frame-coupled overlay work:** use the separate `react-native-vision-camera-realtime` skill. Load the basic frame-processing reference too only when setup or API fundamentals are also needed.
- **New install, getting a Camera on screen, permissions, minimum boilerplate** → [references/quickstart-v5.md](references/quickstart-v5.md)
- **Porting a v4 codebase, understanding what changed** → [references/migration-v4-to-v5.md](references/migration-v4-to-v5.md) (load this FIRST when the user mentions v4, takePhoto, useCameraFormat, format prop, photo/video boolean props, or CodeScanner in core)
- **Porting a whole v4 *screen* — want a complete before/after file to transplant** → [references/migration-templates.md](references/migration-templates.md) (full copy-paste templates: photo screen, video screen, frame-processor+ML, barcode scanner, pro camera)
- **Choosing/attaching outputs, fps/HDR/resolution via constraints, session lifecycle** → [references/outputs-and-constraints.md](references/outputs-and-constraints.md)
- **Frame Processors, worklets, async frame work, pixel formats, writing a native plugin** → [references/frame-processors.md](references/frame-processors.md) (load this when user says "frame processor", "worklet", "ML on frames", "Nitro plugin", "vision-camera-plugin-*")
- **Capturing photos (incl. callbacks, RAW, HDR, preview image), recording video, Recorder lifecycle, manual AE/AF/AWB, exposure bias, zoom, focus** → [references/capture-and-controls.md](references/capture-and-controls.md)
- **Depth streaming, multi-cam, Skia preview, GPU resizer for ML, barcode scanner package, GPS location metadata, custom native outputs** → [references/advanced-features.md](references/advanced-features.md)

When in doubt, load [references/migration-v4-to-v5.md](references/migration-v4-to-v5.md) — it covers the shape of the new API by contrasting it with v4 and is the fastest orientation.

## Non-negotiable rules for v5 code

These are the rules that catch people who "know" v4. Apply them without asking:

1. **Install the Nitro peers.** `react-native-nitro-modules` and `react-native-nitro-image` are required peer deps. Frame processors additionally require `react-native-vision-camera-worklets` AND `react-native-worklets` (Software Mansion's — not `-core`). Worklets - https://docs.swmansion.com/react-native-worklets/docs/
2. **`outputs={[...]}` replaces `photo` / `video` / `frameProcessor` / `codeScanner` props.** Create outputs with `usePhotoOutput`, `useVideoOutput`, `useFrameOutput`, `useDepthOutput`, `useObjectOutput` (or `useBarcodeScannerOutput` from the barcode package) and pass them in an array. Capture methods (`capturePhoto`, `createRecorder`) live on the Output, not the Camera ref.
3. **There is no `format` prop and no `useCameraFormat`.** Use `constraints={[...]}` — array order = priority, descending. The Camera negotiates the closest supported config automatically, so an impossible constraint like `{ fps: 99999 }` never throws.
4. **`takePhoto()` does not exist.** Use `photoOutput.capturePhoto(settings, callbacks)` for in-memory `Photo`, or `photoOutput.capturePhotoToFile(...)` for a file path. The default path is in-memory — do not write temp files unless explicitly asked.
5. **Frame Processor plugins must be Nitro Modules.** The v4 `FrameProcessorPlugin` base class, `VISION_EXPORT_SWIFT_FRAME_PROCESSOR` macro, and `VisionCameraProxy.addFrameProcessorPlugin` are gone. A v5 plugin is a `HybridObject` with a typed Nitro spec. See [references/frame-processors.md](references/frame-processors.md).
6. **Every `Frame` (and `Depth`) MUST be `.dispose()`d.** The buffer pool is bounded; leaking a frame stalls the pipeline. Wrap work in `try { ... } finally { frame.dispose() }`. When offloading via `asyncRunner.runAsync(...)`, dispose inside the async callback if it returned `true`, and dispose immediately in the `else` branch when it returned `false`.
7. **CodeScanner is not in core.** `react-native-vision-camera-barcode-scanner` is a separate package, MLKit-based on both platforms. For iOS-only object detection (QR, faces, bodies via native AVFoundation metadata, no ML dep), use `useObjectOutput` from core.
8. **Keep the Camera mounted; toggle `isActive`.** Remounting tears down the session. Integrate with `useIsFocused()` from react-navigation so the session goes Idle → Ready while not on screen, and keeps preferences warm for fast resume.
9. **Frame output `pixelFormat` defaults to `'native'` (zero-copy), NOT `'yuv'`.** `'native'` streams in the session's negotiated `nativePixelFormat` with zero conversions (it may resolve to a YUV, RGB, RAW, or `'private'` format; verify the actual one via `frame.pixelFormat`). `'yuv'` picks the YUV format closest to native and is the best general-purpose CPU-accessible choice (MLKit/OpenCV/Skia); `'rgb'` forces a YUV-to-RGB conversion with about 2.6 times more bandwidth, so use it only when a consumer hard-requires RGB. `useDepthOutput` has **no** `pixelFormat` option. For ML consumers that require CPU-visible RGB or tensor input, prefer `react-native-vision-camera-resizer` over paying a per-frame RGB conversion in the Camera pipeline.
<!-- source: useFrameOutput.ts:123 (`pixelFormat = 'native'` default); VideoPixelFormat.ts:52-62; CameraFrameOutput.nitro.ts:71,84-86 ("recommended to use 'native' ... zero-copy GPU-only path"); useDepthOutput.ts:70-77 (options have no pixelFormat) -->

10. **Do not hand-clamp FPS/resolution with `Math.min/Math.max`.** That was a v4 workaround. In v5 the Constraints API negotiates internally — express intent and let the Camera pick.
11. **Worklets mutate Reanimated SharedValues directly in v5.** This is suitable for ordinary asynchronous UI or animation state. For frame-locked overlays, use the separate real-time skill and draw from the same frame with Skia or WebGPU.

## Operating rules for this skill

- Never invent v4→v5 API shapes. If a v4 API has no documented v5 equivalent in the references, say so and link to [the v5 docs](https://visioncamera.margelo.com) — do not guess.
- Do not add documentation files (README, CHANGELOG) unless the user asks.
- Assume the user is on v5 unless they show v4 code. If they show v4 code, load [references/migration-v4-to-v5.md](references/migration-v4-to-v5.md) before writing anything.
- When writing a new Camera example, default to the hook-based declarative form (`useCameraPermission` + `useCameraDevice` + `usePhotoOutput` + `<Camera />`). Use the imperative `VisionCamera.createCameraSession(...)` API only when the user asks for multi-cam or full programmatic control.
- For a basic ML path whose consumer needs CPU-visible input, recommend `react-native-vision-camera-resizer` over the v4-era `vision-camera-resize-plugin`. Route latency-critical ML, GPU inference, and live overlays to `react-native-vision-camera-realtime` instead of prescribing the Resizer universally.
<!-- source: react-native-vision-camera-resizer/src/specs/GPUFrame.nitro.ts + Resizer.nitro.ts (GPU resize→GPUFrame). The often-quoted "~5×" figure is a blog claim, not in source, so it is omitted here. -->

- Verify peer dependency installs. A user reporting a native crash after install 95% of the time has missed `react-native-nitro-modules`, `react-native-nitro-image`, or (for frame processors) `react-native-worklets` + `react-native-vision-camera-worklets`.

## Authoritative links

- Docs: https://visioncamera.margelo.com
- `llms.txt` index: https://visioncamera.margelo.com/llms.txt
- V5 release notes (includes migration snippets): `gh api repos/mrousavy/react-native-vision-camera/releases/tags/v5.0.0`
- Blog announcement: https://blog.margelo.com/whats-new-in-visioncamera-v5
- Main repo: https://github.com/mrousavy/react-native-vision-camera
- V4 snapshot (archived docs): https://visioncamera4.margelo.com
