---
name: optimize-react-native
description: Performance rules for React Native apps. Use for any React Native performance problem, review, or optimization task - dropped frames, janky or slow scrolling, slow startup, excessive re-renders, high memory or CPU - and proactively when writing performance-sensitive UI code such as styling, lists, or animations.
license: MIT
metadata:
  author: margelo
  scope: react-native
  tags: react-native, performance, optimization, rendering, lists, scrolling, re-renders, startup, memory, profiling, frame-drops, ios, android
---

# optimize-react-native

Performance guidelines for React Native apps, organized as one reference per topic. Pick the matching topic below and read the reference before writing or reviewing code. Measure before and after every fix — each reference names the right tool for its topic.

## Topics

| Topic | Use when | Reference |
|---|---|---|
| iOS view styling | Shadows (`shadow*` props, `boxShadow`), rounded corners / border radius, `overflow: 'hidden'`, offscreen rendering, janky scrolling on iOS | [references/ios-view-styling.md](./references/ios-view-styling.md) |

If no topic matches, don't invent rules: measure first, then apply the closest reference.
