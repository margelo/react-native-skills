---
id: ios-view-styling
title: iOS view styling — shadows, border radius, clipping
scope: optimize-react-native
keywords: shadow, shadowColor, shadowOffset, shadowOpacity, shadowRadius, boxShadow, borderRadius, overflow hidden, offscreen rendering, CALayer, shadowPath, CAShapeLayer, mask, clipsToBounds, list scroll performance
---

# iOS view styling — shadows, border radius, clipping

Every rule here is about avoiding **offscreen render passes**: Core Animation rendering a layer subtree into a separate GPU buffer before compositing it. One offscreen pass is invisible; the same styling repeated in every list row is dropped frames while scrolling.

These rules target the iOS renderer. Android has different mechanics (`elevation`, the Android view system) and will get its own reference when added.

## Mental model

Fabric maps every `<View>` to a `CALayer` and picks, per view, between a cheap Core Animation path and an expensive fallback:

| Styling | Fast path | Slow path (offscreen pass) |
|---|---|---|
| `shadow*` props | Opaque `backgroundColor` on the same view → precomputed `CALayer.shadowPath` | No/translucent background → Core Animation reads the layer's alpha channel offscreen to derive the shadow shape |
| `boxShadow` | — (none) | Always adds an extra layer with a `CALayer.mask` |
| Border radius | All four corners equal → `CALayer.cornerRadius` | Unequal corners → background moves to a sublayer clipped by `CALayer.mask = CAShapeLayer` |
| `overflow: 'hidden'` | — | `UIView.clipsToBounds`; combined with unequal corners it masks the entire subtree |

To see the cost, run the app and enable **Debug → Color Off-Screen Rendered** in the iOS Simulator (or the same option in Instruments' Core Animation template). Offscreen-rendered regions are tinted yellow — after applying these rules the tint should disappear.

## 1. Shadows need an opaque background on the same view

The `shadow*` props only hit the fast path when the view they are set on has a **fully opaque** `backgroundColor`. React Native then precomputes `CALayer.shadowPath` from the view's rounded rect and Core Animation never has to inspect pixels. Without an opaque background, the shadow is derived from the layer's alpha channel in an offscreen pass, and dev builds warn: *"has a shadow set but cannot calculate shadow efficiently"*.

Putting the background on a child does not help — the check is per-view.

```tsx
// Avoid: shadow view has no background; the opaque color lives on a child.
<View style={{ shadowColor: '#000', shadowOpacity: 0.25, shadowRadius: 8, shadowOffset: { width: 0, height: 4 } }}>
  <View style={{ backgroundColor: 'red', borderRadius: 12 }}>{content}</View>
</View>

// Prefer: opaque background and shadow on the same view.
<View
  style={{
    backgroundColor: 'red',
    borderRadius: 12,
    shadowColor: '#000',
    shadowOpacity: 0.25,
    shadowRadius: 8,
    shadowOffset: { width: 0, height: 4 },
  }}>
  {content}
</View>
```

### Translucent backgrounds: flatten them

A translucent background usually sits on top of a known opaque color. Flatten the two into one opaque color (`result = alpha × top + (1 − alpha) × bottom`, per channel) and use that:

```tsx
// Avoid: rgba background forces the slow alpha-channel shadow.
// Card sits on an opaque #F2F2F7 screen background.
<View style={{ backgroundColor: 'rgba(255,255,255,0.85)', ...shadowStyle }} />

// Prefer: 85% white flattened over #F2F2F7 → #FDFDFE, visually identical and opaque.
<View style={{ backgroundColor: '#FDFDFE', ...shadowStyle }} />
```

This only works when the backdrop is a solid color. If the view floats over an image or video and must stay translucent, move the shadow to a different, opaque view instead of paying the offscreen pass.

## 2. Prefer the `shadow*` props over `boxShadow`

Even though `shadow*` props are dubbed "legacy", prefer them on iOS. `boxShadow` renders each shadow into an extra `CALayer` that **always** carries a `CAShapeLayer` mask (to punch the view's own region out of the shadow) — an unconditional offscreen pass, no matter how the view is styled. The `shadow*` props can use the `shadowPath` fast path from rule 1.

```tsx
// Avoid (iOS): always masks, always offscreen.
<View style={{ backgroundColor: '#fff', borderRadius: 12, boxShadow: '0 4px 12px rgba(0,0,0,0.25)' }} />

// Prefer: eligible for the shadowPath fast path.
<View
  style={{
    backgroundColor: '#fff',
    borderRadius: 12,
    shadowColor: '#000',
    shadowOpacity: 0.25,
    shadowRadius: 6, // to match a boxShadow blur, halve it: RN itself maps boxShadow blur → shadowRadius = blur / 2
    shadowOffset: { width: 0, height: 4 },
  }} />
```

`boxShadow` is the only option for spread, `inset`, or multiple shadows, and it behaves identically on Android. When you genuinely need those features, use it — but keep it off views that repeat in lists.

## 3. Keep the border radius uniform

The `CALayer.cornerRadius` fast path requires **all four corners to have the same circular radius**. The moment any corner differs — including a corner left at `0`, or an elliptical radius from a percentage value on a non-square view — the background is moved into a sublayer clipped by `CALayer.mask = CAShapeLayer`, and borders are CPU-drawn into a separate layer. That is an offscreen pass per view, per frame.

```tsx
// Avoid: unequal corners → CAShapeLayer mask.
<View
  style={{
    borderTopLeftRadius: 10,
    borderBottomLeftRadius: 10,
    borderTopRightRadius: 20,
    borderBottomRightRadius: 20,
    backgroundColor: '#1C1C1E',
  }} />
```

If this shape appears once, accept it. If it repeats — list rows, chat bubbles, segmented cards — split it into **two views that each use one uniform `borderRadius`** and overlap them so each view's unwanted rounded corners are hidden under its sibling's opaque body. Both views then use plain `cornerRadius` and no masks exist anywhere.

The overlap must be at least the **sum of the two radius values**; anything less leaves sliver gaps at the seam where both corner curves cut away.

```tsx
function Row({ children }: { children: React.ReactNode }) {
  return (
    <View style={styles.row}>
      {/* Background split: left corners r=10, right corners r=20 */}
      <View style={styles.background} pointerEvents="none">
        <View style={styles.backgroundLeft} />
        <View style={styles.backgroundRight} />
      </View>
      <View style={styles.content}>{children}</View>
    </View>
  )
}

const styles = StyleSheet.create({
  row: { height: 64 },
  background: { ...StyleSheet.absoluteFillObject, flexDirection: 'row' },
  backgroundLeft: {
    flex: 1,
    borderRadius: 10,
    backgroundColor: '#1C1C1E',
    // Overlap the sibling by leftRadius + rightRadius (10 + 20). The sibling's
    // opaque body covers this view's rounded right corners, and vice versa.
    marginRight: -30,
  },
  backgroundRight: {
    flex: 1,
    borderRadius: 20,
    backgroundColor: '#1C1C1E',
  },
  content: { flex: 1, flexDirection: 'row', alignItems: 'center', paddingHorizontal: 16 },
})
```

Constraints of this pattern: both views need the same opaque background color, and the later sibling must be the one on top (default sibling order — no `zIndex` needed). It spends one extra view to remove a mask, which is a clear win when the style is repeated; verify with the offscreen-render coloring rather than applying it blindly to one-off views.

## 4. Treat `overflow: 'hidden'` as a last resort

`overflow: 'hidden'` sets `UIView.clipsToBounds`. With rounded corners this forces the clip to follow the border-radius path — with unequal corners that is a `CAShapeLayer` mask over the **entire subtree**, and image children get masks of their own. Clipping is re-evaluated every frame, so a decorative `overflow: 'hidden'` on a list row is one of the most expensive habits in RN styling.

Do not add it protectively "so nothing pokes out". Add it only when content genuinely must be cut off (reveal animations, progress bars, parallax crops) and no restructuring can avoid it.

The most common misuse is rounding children the lazy way:

```tsx
// Avoid: clipping the whole card just to round the image's corners.
<View style={{ borderRadius: 20, overflow: 'hidden' }}>
  <Image source={cover} style={StyleSheet.absoluteFillObject} />
  <Text style={styles.title}>{title}</Text>
</View>

// Prefer: pass the radius to the children that actually touch the corners.
<View style={{ borderRadius: 20 }}>
  <Image source={cover} style={[StyleSheet.absoluteFillObject, { borderRadius: 20 }]} />
  <Text style={styles.title}>{title}</Text>
</View>
```

If a child touches only some corners (e.g. a header image needing just the top two rounded), rounding those two corners on the child means unequal corners (rule 3) — but a mask confined to one image layer is still far cheaper than `clipsToBounds` masking the whole card. Prefer, in order: uniform radius on the child → non-uniform radius on the smallest child that needs it → `overflow: 'hidden'` only if neither works.

## Review checklist

When reviewing styles on iOS-visible views, flag:

- `shadowColor`/`shadowOpacity` on a view without an opaque `backgroundColor` (rule 1) — also watch for the runtime warning in Metro logs.
- Any `boxShadow` on views inside lists (rule 2).
- `borderTopLeftRadius` / `borderTopRightRadius` / `borderBottomLeftRadius` / `borderBottomRightRadius` producing unequal corners on repeated views (rule 3).
- `overflow: 'hidden'` combined with `borderRadius` (rule 4).

Then confirm the fix with **Simulator → Debug → Color Off-Screen Rendered**: yellow regions that disappear are offscreen passes you removed.
