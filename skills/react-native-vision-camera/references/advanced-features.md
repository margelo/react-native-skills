# Advanced features

Depth, multi-cam, Skia previews, the GPU Resizer, barcode scanning (separate package), GPS/location metadata, and writing a custom native output.

## Depth frames (LiDAR / ToF / disparity)

```tsx
import { Camera, useDepthOutput, useCameraDevice } from 'react-native-vision-camera'

const device = useCameraDevice('back')
const depthOutput = useDepthOutput({
  // useDepthOutput options: targetResolution, enableFiltering, dropFramesWhileBusy, allowDeferredStart, onDepth, onDepthFrameDropped.
  // There is NO pixelFormat option — read the resolved format from `depth.pixelFormat` (a DepthPixelFormat):
  //   'depth-16-bit' | 'depth-32-bit' | 'depth-point-cloud-32-bit' | 'disparity-16-bit' | 'disparity-32-bit' | 'unknown'
  // source: useDepthOutput.ts:70-77 (options); DepthPixelFormat.ts:18-24
  onDepth(depth) {
    'worklet'
    try {
      // depth.width, depth.height, depth.pixelFormat, depth.orientation, depth.timestamp
      // depth.depthDataAccuracy, depth.depthDataQuality, depth.isDepthDataFiltered
      // depth.isMirrored, depth.isValid, depth.bytesPerRow, depth.cameraCalibrationData (iOS)
      // depth.getDepthData(), depth.getNativeBuffer()
      // depth.convert('depth-32-bit') / depth.convertAsync(...)
    } finally {
      depth.dispose() // REQUIRED — same buffer-pool rule as Frame
    }
  },
})

<Camera device={device} isActive={true} outputs={[depthOutput]} />
```

- Requires `react-native-vision-camera-worklets` + `react-native-worklets`.
- Not every device exposes depth. There is **no** `supportsDepthCapture` flag — gate on `device.mediaTypes.includes('depth')`; depth-capable virtual cameras report `['video', 'depth']`. <!-- source: CameraDevice.nitro.ts:386-388 (`readonly mediaTypes: MediaType[]`); no supportsDepthCapture exists -->.
- Two sources:
  - LiDAR / ToF / Infrared → true depth frames (`depth-16-bit`, `depth-32-bit`, `depth-point-cloud-32-bit`).
  - Dual or triple virtual cameras → disparity frames synthesized from stereo (`disparity-16-bit`, `disparity-32-bit`).
- `depth.convert(target)` / `depth.convertAsync(target)` return a **new** derivative `Depth` in the target format (must be one of `depth.availableDepthPixelFormats`); the original is untouched. <!-- source: Depth.nitro.ts:194,201,133 -->
- Native plugins: same Nitro pattern as Frame, but use the `Depth` spec type and cast to `NativeDepth` (iOS: `AVDepthData`).

## Multi-camera sessions

Front + back simultaneously (or any combination the device supports). iOS 13+ and supported Android devices only.

```ts
// Probe first — multi-cam is a platform-level capability, not a CameraDevice flag:
if (!VisionCamera.supportsMultiCamSessions) return
// then pick a valid input pair from `deviceFactory.supportedMultiCamDeviceCombinations`
// source: CameraFactory.nitro.ts:71; CameraSession.nitro.ts:70-74,182-186

// Imperative only — no declarative shorthand for multi-cam.
const session = await VisionCamera.createCameraSession(/* isMultiCam */ true)

const backDevice = await getDefaultCameraDevice('back')
const frontDevice = await getDefaultCameraDevice('front')

const backPreview = VisionCamera.createPreviewOutput()  // takes no args — source: CameraFactory.nitro.ts:186
const frontPreview = VisionCamera.createPreviewOutput()
const backVideo = VisionCamera.createVideoOutput({ enableAudio: true })

const [backController, frontController] = await session.configure([
  {
    input: backDevice,
    outputs: [
      { output: backPreview,  mirrorMode: 'auto' },
      { output: backVideo,    mirrorMode: 'auto' },
    ],
    constraints: [{ fps: 30 }],
  },
  {
    input: frontDevice,
    outputs: [{ output: frontPreview, mirrorMode: 'auto' }],
    constraints: [],
  },
], {})

await session.start()
```

Render each preview with `<NativePreviewView />`, bound to its own `CameraPreviewOutput`. For picture-in-picture UX, render the front preview as a smaller absolutely-positioned view on top.

## Skia previews — shaders and live effects

```tsx
import { SkiaCamera } from 'react-native-vision-camera-skia'

<SkiaCamera
  device={device}
  isActive={true}
  pixelFormat="yuv"
  onFrame={(frame, render) => {
    'worklet'
    render(({ frameTexture, canvas }) => {
      canvas.drawImage(frameTexture, 0, 0)
      // draw overlays, apply a Skia ImageFilter, etc.
    })
    frame.dispose()
  }}
/>
```

- `<SkiaCamera />` replaces `<Camera />`'s preview with a Skia canvas and **always** attaches a frame output (you can't opt out).
- Pixel formats: `'native'` / `'yuv'` / `'rgb'`. The SkiaCamera default is `'yuv'` on iOS and `'native'` on Android. With `'native'`, verify Skia compatibility (RAW or `'private'` may not be supported). <!-- source: SkiaCamera.tsx:200-203 (DEFAULT_PIXEL_FORMAT Platform.select), :170 -->
- `focusTo` works with SkiaCamera in v5.
- For manual rendering without the wrapper: use `NativeBuffer` + Skia's `MakeImageFromNativeBuffer()`.
- Peer deps: `@shopify/react-native-skia`.

## GPU Resizer — the ML fast-path

`react-native-vision-camera-resizer` is Margelo's GPU-accelerated replacement for the CPU-based `vision-camera-resize-plugin`. It runs on Metal (iOS) and Vulkan + `AHardwareBuffer` (newer Android versions), and returns a pooled `GPUFrame`. <!-- source: ResizerFactory.nitro.ts:83-87 (Metal/Vulkan, isAvailable()); GPUFrame.nitro.ts. The often-quoted "~5×" multiplier is a blog claim, not in source. -->

```ts
import { useResizer } from 'react-native-vision-camera-resizer'

// useResizer returns { state: 'loading' | 'ready' | 'error', resizer, error }.
// There is NO isResizerAvailable() export — gate on `state` / `resizer != null`, and inspect `error`.
const { resizer } = useResizer({
  width: 128,
  height: 128,
  channelOrder: 'rgb',     // 'rgb' | 'bgr'
  dataType: 'float32',     // 'int8' | 'uint8' | 'float16' | 'float32'
  scaleMode: 'cover',      // 'cover' | 'contain'
  pixelLayout: 'planar',   // 'planar' = NCHW ([1,3,H,W]); 'interleaved' = NHWC ([1,H,W,3])
})

const frameOutput = useFrameOutput({
  pixelFormat: 'yuv', // resizer is happiest with YUV input
  onFrame(frame) {
    'worklet'
    if (resizer == null) { frame.dispose(); return } // still loading or errored
    const resized = resizer.resize(frame)
    const pixels = resized.getPixelBuffer() // native buffer ready for ONNX/TFLite
    try { /* model.run(pixels) */ }
    finally {
      resized.dispose()
      frame.dispose()
    }
  },
})
```
<!-- source: react-native-vision-camera-resizer/src/index.ts (no isResizerAvailable export); useResizer.ts:13-16,55-94 (ResizerState + guarded example); OutputFormat.ts (ChannelOrder='rgb'|'bgr', DataType, PixelLayout); ResizerFactory.nitro.ts:16,50 (ScaleMode) -->

Gate on the hook's `state` / `resizer` (there is no `isResizerAvailable()`; the underlying capability check is `ResizerFactory.isAvailable()`). Provide a CPU fallback when `state === 'error'`. Dispose the returned `GPUFrame` like a regular Frame — it's a pooled GPU resource. <!-- source: useResizer.ts:13-16; ResizerFactory.nitro.ts:87 (isAvailable); GPUFrame.nitro.ts -->

## Barcode scanner — `react-native-vision-camera-barcode-scanner`

MLKit on both platforms, so format behavior matches across iOS and Android. The classic v4 `'ean-13' ↔ UPC-A` iOS quirk is gone.

### Easiest — drop-in view

```tsx
import { CodeScanner } from 'react-native-vision-camera-barcode-scanner'

<CodeScanner
  style={{ flex: 1 }} // required — CodeScannerOptions.style is not optional. source: CodeScanner.tsx:20
  isActive
  barcodeFormats={['qr-code', 'ean-13']}
  onBarcodeScanned={(barcodes) => console.log(barcodes[0]?.rawValue)}
  onError={(e) => console.error(e)}
/>
```

### Integrated — Camera output

```tsx
import { useBarcodeScannerOutput } from 'react-native-vision-camera-barcode-scanner'

const barcodeOutput = useBarcodeScannerOutput({
  barcodeFormats: ['qr-code'],
  onBarcodeScanned: (barcodes) => {},
})

<Camera outputs={[photoOutput, barcodeOutput]} /* ... */ />
```

### Frame-processor — full control

```tsx
import { useBarcodeScanner } from 'react-native-vision-camera-barcode-scanner'

const scanner = useBarcodeScanner({ barcodeFormats: ['qr-code'] })
const frameOutput = useFrameOutput({
  onFrame(frame) {
    'worklet'
    try {
      const codes = scanner.scanCodes(frame)
      if (codes.length) found.value = codes
    } finally { frame.dispose() }
  },
})
```

Performance rule: list only the formats you need.

## iOS-only native object output (no ML dep)

If all you need is QR + face + body detection on iOS, skip MLKit and use the native `AVCaptureMetadataOutput` path:

```tsx
import { useObjectOutput, isScannedCode, isScannedFace } from 'react-native-vision-camera'

const objectOutput = useObjectOutput({
  types: ['qr', 'face', 'human-body'],
  onObjectsScanned: (objects) => {
    for (const o of objects) {
      if (isScannedCode(o)) console.log('code:', o.value)
      else if (isScannedFace(o)) console.log('face:', o.faceID)
    }
  },
})

<Camera outputs={[objectOutput]} />
```

Android has no native equivalent — use the MLKit barcode scanner there.

## GPS / location metadata — `react-native-vision-camera-location`

```tsx
import { useLocation } from 'react-native-vision-camera-location'

const loc = useLocation({})
useEffect(() => { if (!loc.hasPermission) loc.requestPermission() }, [loc.hasPermission])

// Attach to photo:
const photo = await photoOutput.capturePhoto({ location: loc.currentLocation }, {})

// Attach to video recorder:
const recorder = await videoOutput.createRecorder({ location: loc.currentLocation })
```

Adds EXIF GPS tags to JPEGs and location metadata to mp4/mov. Imperative variant: `createLocationManager(...)` + `addOnLocationChangedListener`.

## Custom native `CameraOutput` (extensibility)

V5 exposes `NativeCameraOutput` so plugin authors can ship a fully custom output (e.g. a proprietary HDR pipeline, a ML streaming output) as a separate Nitro Module without forking the library.

```ts
// Your spec
export interface MyOutput extends HybridObject<{ ios: 'swift', android: 'kotlin' }> {
  // ... methods + events your output exposes
}
```

Implement `NativeCameraOutput` (iOS) / its Android equivalent, expose a factory function in your Nitro module, and consumers attach via the standard `outputs={[myOutput, ...]}` prop. Delegate scaffolding to the build-nitro-modules skill.

## Constraints for advanced features — quick recap

```ts
// Photo HDR
[{ photoHDR: true }]

// Video HDR 10-bit HLG
[{ videoDynamicRange: { bitDepth: 'hdr-10-bit', colorSpace: 'hlg-bt2020', colorRange: 'full' } }]

// Apple Log
[{ videoDynamicRange: { bitDepth: 'hdr-10-bit', colorSpace: 'apple-log', colorRange: 'full' } }]

// Cinematic stabilization
[{ videoStabilizationMode: 'cinematic-extended' }]

// Prefer binned sensor readout
[{ binned: true }]

// Optimize for the frame output's resolution
[{ resolutionBias: frameOutput }]
```

Always pair advanced features with `device.isSessionConfigSupported(config)` / `onSessionConfigSelected` so your UI reflects what the Camera actually picked. <!-- source: CameraDevice.nitro.ts:611; useCamera.ts:71 -->

## Pointers

- Docs — Depth Output: https://visioncamera.margelo.com/docs/depth-output
- Docs — Object Output (iOS): https://visioncamera.margelo.com/docs/object-output
- API — `Depth`: https://visioncamera.margelo.com/api/react-native-vision-camera/hybrid-objects/Depth
- API — `DepthPixelFormat`: https://visioncamera.margelo.com/api/react-native-vision-camera/type-aliases/DepthPixelFormat
- Core repo: https://github.com/mrousavy/react-native-vision-camera
- Barcode scanner (MLKit, both platforms): https://github.com/mrousavy/react-native-vision-camera/tree/main/packages/react-native-vision-camera-barcode-scanner
- GPU Resizer (Metal on iOS, Vulkan on Android): https://github.com/mrousavy/react-native-vision-camera/tree/main/packages/react-native-vision-camera-resizer
- Skia preview: https://github.com/mrousavy/react-native-vision-camera/tree/main/packages/react-native-vision-camera-skia
- Location (EXIF GPS): https://github.com/mrousavy/react-native-vision-camera/tree/main/packages/react-native-vision-camera-location
- Nitro scaffolding for custom `NativeCameraOutput`: use the `build-nitro-modules` skill.
- Related: [outputs-and-constraints.md](./outputs-and-constraints.md), [frame-processors.md](./frame-processors.md), [capture-and-controls.md](./capture-and-controls.md), [migration-v4-to-v5.md](./migration-v4-to-v5.md)
