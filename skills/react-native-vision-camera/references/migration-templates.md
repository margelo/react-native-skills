# Migration templates — V4 → V5 (full before/after files)

Copy-paste-ready, whole-screen templates for the most common camera screens. Each shows the complete v4 file and its complete v5 equivalent so you can port a screen in one pass. Every v5 template uses only the real v5.0.11 API — each non-obvious surface carries a `// source:` pointer into the cloned library.

Pair this with [migration-v4-to-v5.md](./migration-v4-to-v5.md) for the conceptual mapping and per-API notes. Load this file when you want to *transplant a screen wholesale*, not reason about individual APIs.

Hard rules these templates bake in (don't undo them):

- `capturePhoto(settings, callbacks)` takes **two** arguments — pass `{}` for callbacks if you have none. <!-- source: CameraPhotoOutput.nitro.ts:310 -->
- A `Recorder` is **single-use** — `createRecorder(...)` again for every recording. <!-- source: CameraVideoOutput.nitro.ts:238-243 -->
- Every `Frame`/`Depth` must be `.dispose()`d, even on error paths. <!-- source: Frame.nitro.ts:97; AsyncRunner.ts:18-44 -->
- Keep the Camera mounted; toggle `isActive` (here via `useIsFocused()`). <!-- source: useCamera.ts:40 (isActive) -->
- `useFrameOutput` `pixelFormat` defaults to `'native'`; pass `'yuv'` for CPU/ML access. <!-- source: useFrameOutput.ts:123 -->

---

## Template A — Photo capture screen

### ❌ V4

```tsx
import React, { useEffect, useRef, useState } from 'react'
import { StyleSheet, View, Image, Pressable } from 'react-native'
import {
  Camera,
  useCameraDevice,
  useCameraPermission,
} from 'react-native-vision-camera'

export function PhotoScreen() {
  const camera = useRef<Camera>(null)
  const device = useCameraDevice('back')
  const { hasPermission, requestPermission } = useCameraPermission()
  const [uri, setUri] = useState<string>()

  useEffect(() => { if (!hasPermission) requestPermission() }, [hasPermission])

  const onShutter = async () => {
    const file = await camera.current?.takePhoto({ flash: 'auto' })
    if (file != null) setUri(`file://${file.path}`)
  }

  if (!hasPermission || device == null) return null

  return (
    <View style={StyleSheet.absoluteFill}>
      <Camera ref={camera} style={StyleSheet.absoluteFill} device={device} isActive photo />
      {uri && <Image source={{ uri }} style={styles.preview} />}
      <Pressable style={styles.shutter} onPress={onShutter} />
    </View>
  )
}
```

### ✅ V5

```tsx
import React, { useEffect, useState } from 'react'
import { StyleSheet, View, Pressable } from 'react-native'
import { useIsFocused } from '@react-navigation/native'
import { NitroImage } from 'react-native-nitro-image'
import type { Image } from 'react-native-nitro-image'
import {
  Camera,
  useCameraDevice,
  useCameraPermission,
  usePhotoOutput,
} from 'react-native-vision-camera'

export function PhotoScreen() {
  const device = useCameraDevice('back')
  const { hasPermission, requestPermission } = useCameraPermission()
  const isFocused = useIsFocused()
  const [image, setImage] = useState<Image>()

  // Output is created once and passed via `outputs`. Capture lives on the output, not a ref.
  const photoOutput = usePhotoOutput({ qualityPrioritization: 'balanced' }) // source: usePhotoOutput.ts:31

  useEffect(() => { if (!hasPermission) requestPermission() }, [hasPermission, requestPermission])

  const onShutter = async () => {
    // Two args required: settings, callbacks. Returns an in-memory Photo (no temp file). source: CameraPhotoOutput.nitro.ts:310
    const photo = await photoOutput.capturePhoto({ flashMode: 'auto' }, {})
    const img = await photo.toImageAsync() // source: Photo.nitro.ts:195
    setImage(img)
    photo.dispose() // free native memory. source: Photo.nitro.ts:28-29
  }

  if (!hasPermission || device == null) return null

  return (
    <View style={StyleSheet.absoluteFill}>
      <Camera
        style={StyleSheet.absoluteFill}
        device={device}
        isActive={isFocused}
        outputs={[photoOutput]}
      />
      {image && <NitroImage image={image} style={styles.preview} />}
      <Pressable style={styles.shutter} onPress={onShutter} />
    </View>
  )
}
```

What changed: `photo` boolean → `usePhotoOutput()` in `outputs`; `takePhoto` (Camera ref, wrote a file) → `photoOutput.capturePhoto(settings, {})` (in-memory `Photo`); `flash` → `flashMode`; render via `toImageAsync()` + `<NitroImage />` instead of a `file://` URI; unmount-to-hide → `isActive={isFocused}`.

---

## Template B — Video recording screen

### ❌ V4

```tsx
import React, { useRef, useState } from 'react'
import { StyleSheet, Pressable } from 'react-native'
import { Camera, useCameraDevice } from 'react-native-vision-camera'

export function VideoScreen() {
  const camera = useRef<Camera>(null)
  const device = useCameraDevice('back')
  const [recording, setRecording] = useState(false)

  const start = () => {
    setRecording(true)
    camera.current?.startRecording({
      onRecordingFinished: (video) => console.log('finished', video.path),
      onRecordingError: (e) => console.error(e),
    })
  }
  const stop = async () => {
    await camera.current?.stopRecording()
    setRecording(false)
  }

  if (device == null) return null
  return (
    <>
      <Camera ref={camera} style={StyleSheet.absoluteFill} device={device} isActive video audio />
      <Pressable style={styles.shutter} onPress={recording ? stop : start} />
    </>
  )
}
```

### ✅ V5

```tsx
import React, { useRef, useState } from 'react'
import { StyleSheet, Pressable } from 'react-native'
import { useIsFocused } from '@react-navigation/native'
import {
  Camera,
  useCameraDevice,
  useVideoOutput,
} from 'react-native-vision-camera'
import type { Recorder } from 'react-native-vision-camera'

export function VideoScreen() {
  const device = useCameraDevice('back')
  const isFocused = useIsFocused()
  const recorderRef = useRef<Recorder>(null) // a Recorder is single-use; hold the active one
  const [recording, setRecording] = useState(false)

  const videoOutput = useVideoOutput({ enableAudio: true }) // audio is OFF by default. source: CameraVideoOutput.nitro.ts:36-45

  const start = async () => {
    // Create a fresh Recorder per recording. source: CameraVideoOutput.nitro.ts:243
    const recorder = await videoOutput.createRecorder({})
    recorderRef.current = recorder
    setRecording(true)
    // startRecording(onFinished(filePath, reason), onError, onPaused?, onResumed?). source: Recorder.nitro.ts:80-88
    await recorder.startRecording(
      (filePath, reason) => console.log('finished', filePath, reason),
      (e) => console.error(e),
    )
  }

  const stop = async () => {
    await recorderRef.current?.stopRecording() // resolves immediately; onFinished fires after flush. source: Recorder.nitro.ts:98
    recorderRef.current = null
    setRecording(false)
  }

  if (device == null) return null
  return (
    <>
      <Camera
        style={StyleSheet.absoluteFill}
        device={device}
        isActive={isFocused}
        outputs={[videoOutput]}
      />
      <Pressable style={styles.shutter} onPress={recording ? stop : start} />
    </>
  )
}
```

What changed: `video`/`audio` props → `useVideoOutput({ enableAudio: true })`; recording moved off the Camera ref onto a `Recorder` you create per take; the finished callback now yields `(filePath, reason)` where `reason` is `'stopped' | 'max-duration-reached' | 'max-file-size-reached'`. <!-- source: Recorder.nitro.ts:21-24 -->

---

## Template C — Frame processor with ML + Reanimated overlay

### ❌ V4

```tsx
import { useFrameProcessor, Camera, useCameraDevice } from 'react-native-vision-camera'
import { useSharedValue } from 'react-native-reanimated'
import { runAtTargetFps } from 'react-native-vision-camera'

export function ScanScreen() {
  const device = useCameraDevice('back')
  const boxes = useSharedValue([])

  const frameProcessor = useFrameProcessor((frame) => {
    'worklet'
    runAtTargetFps(5, () => {
      'worklet'
      boxes.value = detectObjects(frame) // a v4 plugin
    })
  }, [])

  if (device == null) return null
  return <Camera style={{ flex: 1 }} device={device} isActive frameProcessor={frameProcessor} pixelFormat="yuv" />
}
```

### ✅ V5

```tsx
import React from 'react'
import { StyleSheet } from 'react-native'
import { useIsFocused } from '@react-navigation/native'
import { useSharedValue } from 'react-native-reanimated'
import {
  Camera,
  useCameraDevice,
  useFrameOutput,
  useAsyncRunner,
  CommonResolutions,
} from 'react-native-vision-camera'

export function ScanScreen() {
  const device = useCameraDevice('back')
  const isFocused = useIsFocused()
  const boxes = useSharedValue<Box[]>([])
  const asyncRunner = useAsyncRunner() // dedicated worklet runtime for heavy work. source: useAsyncRunner.ts:33

  const frameOutput = useFrameOutput({
    pixelFormat: 'yuv', // CPU-accessible; default is 'native'. source: useFrameOutput.ts:123
    targetResolution: CommonResolutions.VGA_16_9, // stream small for ML
    onFrame(frame) {
      'worklet'
      // runAtTargetFps is gone — offload heavy work and let backpressure throttle. source: grep "runAtTargetFps" → absent
      const accepted = asyncRunner.runAsync(() => {
        'worklet'
        try {
          boxes.value = detectObjects(frame) // worklets mutate SharedValues directly — no runOnJS
        } finally {
          frame.dispose() // dispose INSIDE the async task when accepted. source: AsyncRunner.ts:26-41
        }
      })
      if (!accepted) frame.dispose() // runner busy → drop this frame, still dispose
    },
  })

  if (device == null) return null
  return (
    <Camera
      style={StyleSheet.absoluteFill}
      device={device}
      isActive={isFocused}
      outputs={[frameOutput]}
    />
  )
}
```

What changed: `frameProcessor={useFrameProcessor(...)}` → `useFrameOutput({ onFrame })` in `outputs`; `pixelFormat` moved from the Camera onto the output (default is now `'native'`); `runAtTargetFps` removed — offload via `useAsyncRunner` with the explicit `accepted ? dispose-inside : dispose-immediate` pattern; native plugins must be Nitro `HybridObject`s. Requires `react-native-vision-camera-worklets` + `react-native-worklets`. <!-- source: useFrameOutput.ts:67-70 -->

---

## Template D — QR / barcode scanner screen

### ❌ V4

```tsx
import { Camera, useCameraDevice, useCodeScanner } from 'react-native-vision-camera'

export function ScannerScreen() {
  const device = useCameraDevice('back')
  const codeScanner = useCodeScanner({
    codeTypes: ['qr', 'ean-13'],
    onCodeScanned: (codes) => console.log(codes[0]?.value),
  })

  if (device == null) return null
  return <Camera style={{ flex: 1 }} device={device} isActive codeScanner={codeScanner} />
}
```

### ✅ V5 — drop-in view (simplest)

```tsx
import { CodeScanner } from 'react-native-vision-camera-barcode-scanner'

export function ScannerScreen() {
  return (
    <CodeScanner
      style={{ flex: 1 }} // CodeScannerOptions requires `style`. source: CodeScanner.tsx:15-30
      isActive
      barcodeFormats={['qr-code', 'ean-13']} // TargetBarcodeFormat values. source: BarcodeFormat.ts:9-31
      onBarcodeScanned={(barcodes) => console.log(barcodes[0]?.rawValue)} // source: Barcode.nitro.ts:64
      onError={(e) => console.error(e)}
    />
  )
}
```

### ✅ V5 — integrated output (when you also need photo/video on the same Camera)

```tsx
import React from 'react'
import { StyleSheet } from 'react-native'
import { useIsFocused } from '@react-navigation/native'
import { Camera, useCameraDevice } from 'react-native-vision-camera'
import { useBarcodeScannerOutput } from 'react-native-vision-camera-barcode-scanner'

export function ScannerScreen() {
  const device = useCameraDevice('back')
  const isFocused = useIsFocused()

  const barcodeOutput = useBarcodeScannerOutput({
    barcodeFormats: ['qr-code'],
    onBarcodeScanned: (barcodes) => console.log(barcodes[0]?.rawValue),
    onError: (e) => console.error(e),
  }) // source: useBarcodeScannerOutput.ts:59-64

  if (device == null) return null
  return (
    <Camera
      style={StyleSheet.absoluteFill}
      device={device}
      isActive={isFocused}
      outputs={[barcodeOutput]}
    />
  )
}
```

What changed: `useCodeScanner` + `codeScanner` prop (in v4 core) → the separate `react-native-vision-camera-barcode-scanner` package (MLKit on both platforms). Code type names changed (`'qr'` → `'qr-code'`); the value field is `rawValue` (was `value`). For iOS-only QR/face/body detection without an ML dependency, use core `useObjectOutput({ types, onObjectsScanned })` + `isScannedCode`/`isScannedFace` instead. <!-- source: ScannedObject.nitro.ts:35-63; isScannedObject.ts:12,21 -->

---

## Template E — "Pro" camera (photo + zoom gesture + tap-to-focus + flash + HDR), V5 only

A kitchen-sink screen showing how v5 composes outputs, constraints, animated values, and the `CameraRef`. There is no single v4 equivalent — in v4 this required `useCameraFormat`, `Reanimated.createAnimatedComponent`, `addWhitelistedNativeProps`, and manual `Gesture` wiring.

```tsx
import React, { useEffect, useRef, useState } from 'react'
import { StyleSheet, Pressable } from 'react-native'
import { useIsFocused } from '@react-navigation/native'
import { useSharedValue } from 'react-native-reanimated'
import {
  Camera,
  useCameraDevice,
  useCameraPermission,
  usePhotoOutput,
  type CameraRef,
} from 'react-native-vision-camera'

export function ProCameraScreen() {
  const device = useCameraDevice('back')
  const { hasPermission, requestPermission } = useCameraPermission()
  const isFocused = useIsFocused()
  const cameraRef = useRef<CameraRef>(null) // source: Camera.tsx:35 (CameraRef)
  const zoom = useSharedValue(1) // 1 = natural default; SharedValue is accepted directly. source: Camera.tsx:120
  const [flashOn, setFlashOn] = useState(false)

  const photoOutput = usePhotoOutput({ previewImageTargetSize: { width: 120, height: 160 } })

  useEffect(() => { if (!hasPermission) requestPermission() }, [hasPermission, requestPermission])

  const onShutter = async () => {
    const photo = await photoOutput.capturePhoto(
      { flashMode: flashOn ? 'on' : 'off' },
      { onPreviewImageAvailable: (thumb) => {/* show thumb instantly */} }, // source: CameraPhotoOutput.nitro.ts:121
    )
    // ...use photo.toImageAsync()...
    photo.dispose()
  }

  const onTapFocus = async (x: number, y: number) => {
    // CameraRef converts view-point → camera-point for you. source: Camera.tsx:199-208
    await cameraRef.current?.focusTo({ x, y })
  }

  if (!hasPermission || device == null) return null

  return (
    <>
      <Camera
        ref={cameraRef}
        style={StyleSheet.absoluteFill}
        device={device}
        isActive={isFocused}
        outputs={[photoOutput]}
        zoom={zoom}                          // animate zoom via Reanimated SharedValue
        enableNativeTapToFocusGesture        // or wire your own gesture → cameraRef.focusTo(...)
        constraints={[{ photoHDR: true }]}   // negotiated intent; never throws. source: Constraint.ts:119
      />
      <Pressable style={styles.flash} onPress={() => setFlashOn((v) => !v)} />
      <Pressable style={styles.shutter} onPress={onShutter} />
    </>
  )
}
```

Key v5 building blocks used: `outputs={[...]}` (not boolean props), `zoom={SharedValue}` (no `createAnimatedComponent`/`addWhitelistedNativeProps`), `enableNativeTapToFocusGesture` (or `cameraRef.focusTo({x,y})`), `constraints={[{ photoHDR: true }]}` (no `useCameraFormat`), and `previewImageTargetSize` + `onPreviewImageAvailable` for an instant thumbnail.

---

## Shared styles (for the templates above)

```ts
const styles = StyleSheet.create({
  preview: { position: 'absolute', bottom: 24, right: 24, width: 96, height: 128, borderRadius: 8 },
  shutter: { position: 'absolute', alignSelf: 'center', bottom: 48, width: 72, height: 72, borderRadius: 36, backgroundColor: 'white' },
  flash: { position: 'absolute', top: 48, right: 24, width: 44, height: 44, borderRadius: 22, backgroundColor: '#0008' },
})
```

## Pointers

- Conceptual mapping + per-API notes: [migration-v4-to-v5.md](./migration-v4-to-v5.md)
- Outputs, constraints, devices, session lifecycle: [outputs-and-constraints.md](./outputs-and-constraints.md)
- Capture & controls (photo/video/zoom/focus/3A): [capture-and-controls.md](./capture-and-controls.md)
- Frame processors & async: [frame-processors.md](./frame-processors.md)
- Barcode / Depth / Skia / Resizer / Location: [advanced-features.md](./advanced-features.md)
