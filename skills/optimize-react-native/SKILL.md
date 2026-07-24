---
name: optimize-react-native
description: Performance rules for React Native apps. Use for any React Native performance problem or review - dropped frames, janky or slow list scrolling, slow startup, excessive re-renders, high memory or CPU, offscreen rendering - and proactively when writing code that commonly causes these, such as styles with shadows (shadow* props or boxShadow), rounded corners / border radius, overflow hidden, or views repeated in lists.
license: MIT
metadata:
  author: margelo
  scope: react-native
  tags: react-native, performance, optimization, rendering, lists, scrolling, re-renders, startup, memory, profiling, frame-drops, ios, android
---

# optimize-react-native

Performance guidelines for React Native apps — any performance issue, not just one subsystem: rendering, lists, re-renders, startup, memory. The skill is organized as one reference per topic; load the matching `references/*.md` file before writing or reviewing code. Each reference explains the underlying mechanics, gives Avoid/Prefer examples, and ends with a review checklist.

## Mental model

A React Native frame crosses three layers — JS (product code, React reconciliation), the framework (Fabric/Shadow tree, layout), and the native renderer (CALayer on iOS, the Android view system). A performance problem lives in one of these layers, and the fix looks completely different depending on which one: memoization won't help a view that forces offscreen rendering, and styling changes won't help a component that re-renders 60 times per second. Identify the layer first — measure, don't guess — then load the topic reference for it.

A recurring theme across all topics: cost is multiplied by repetition. A pattern that is harmless on one screen-level view (an extra render, a mask, a deep tree) becomes dropped frames when it sits inside every list row, chat bubble, card, or grid cell. Review repeated views with much stricter standards than one-offs.

## Routing table — problem to reference

| User is asking about… | Read |
|---|---|
| Shadows (`shadowColor`, `shadowOffset`, `shadowOpacity`, `shadowRadius`, `boxShadow`), rounded corners (`borderRadius`, per-corner values), `overflow: 'hidden'`, clipping, offscreen rendering, the "cannot calculate shadow efficiently" warning, janky list scrolling on iOS | [`references/ios-view-styling.md`](./references/ios-view-styling.md) |

More topics (lists and virtualization, re-renders, startup time, images) will be added as further references — one row and one file each.

If the question is performance-related but doesn't match a row, don't invent rules: measure first (profiler, offscreen-render coloring, React DevTools), identify the layer, and apply the closest reference. View styling is a common hidden culprit, so [`references/ios-view-styling.md`](./references/ios-view-styling.md) is worth a scan even for vague "the app feels slow" reports on iOS.

## Red flags — load the matching reference when you see these in code

- `shadowColor` on a view with no `backgroundColor`, or a translucent (`rgba`/`transparent`) one
- `boxShadow` on views rendered inside lists
- `borderTopLeftRadius` / `borderBottomRightRadius` etc. with unequal values on repeated views
- `overflow: 'hidden'` paired with `borderRadius`

## Verifying fixes

Don't reason about performance from code alone — every fix should be confirmed with a measurement, and each reference names the right tool for its topic. For the current rendering rules: in the iOS Simulator enable **Debug → Color Off-Screen Rendered** (also available in Instruments' Core Animation template); offscreen-rendered regions are tinted yellow, so apply a rule, re-check, and confirm the tint is gone.

## References

| File | Description |
|------|-------------|
| [ios-view-styling.md](./references/ios-view-styling.md) | iOS view-styling fast paths: opaque backgrounds for `shadow*`, `shadow*` vs `boxShadow`, uniform border radius (incl. the two-view split pattern for mixed corner values), `overflow: 'hidden'` as a last resort |
