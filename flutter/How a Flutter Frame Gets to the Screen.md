---
share_link: https://share.note.sx/accpdkjj#fteT+W4RojhWTKn9XyHU1nokRplz92SINiRPhyIMpWQ
share_updated: 2026-09-23T05:05:56+03:00
---
# How a Flutter Frame Gets to the Screen

> [!NOTE]
> **Frame budget: ~16.7 ms at 60 FPS** (less at 90/120 Hz).
> Every frame passes through three stages — UI thread → Raster thread → GPU — before it reaches the display. If any stage takes too long, the frame misses its deadline and you get **jank**.

---

## 🧵 1. UI Thread — *"What should this frame look like?"*

**Runs on:** the Dart/Framework task runner (the *UI isolate* — not an OS thread you manage manually)

**Responsibilities:**

- Executes your Dart code and the Flutter framework itself
- `build()`, `setState()`, state management callbacks
- Layout (measuring and positioning widgets)
- Painting instructions (what to draw, not pixels yet)
- Diffs the widget tree and flattens it into a **Layer Tree**

> [!IMPORTANT]
> The UI thread **never touches pixels**. It only describes *what* needs to be drawn.

### The Layer Tree

A lightweight, compositing description of the scene — no pixels, just metadata:

| Property | Examples |
|---|---|
| Geometry | Position, size, transforms |
| Visual | Opacity, clipping, blur |
| Special | Platform views, pictures |

**Output:** the Layer Tree is handed off to the Raster thread.

⬇️

---

## 🧵 2. Raster Thread — *"Turn that description into pixels"*

**Runs on:** the Raster/GPU task runner

**Responsibilities:**

- Takes the Layer Tree and converts it into actual GPU draw commands
- Rasterizes the scene using **Skia** or **Impeller**
- Renders into a framebuffer/surface ready to be presented

**Common causes of Raster-thread jank:**

- Large or uncached images
- Complex clipping or excessive shadows/blur
- Expensive or uncached shaders
- Very complex scenes (many layers, overdraw)

> [!IMPORTANT] 
> **Skia vs. Impeller:** Skia compiled shaders the first time they were used at runtime, causing visible one-off hitches. **Impeller** precompiles shaders ahead of time, largely eliminating that class of jank.

⬇️

---

## 🖥️ 3. GPU / Display System — *"Show it"*

*Hardware stage*

Executes the rasterized draw commands. The platform's compositor then presents the finished frame on screen — completing the frame within its budget.

---

## Where Jank Actually Comes From

| Symptom | Cause | Fix direction |
|---|---|---|
| **UI-thread jank** | Too slow to *produce* the frame — slow `build()`, heavy layout, expensive Dart computation | Simplify rebuilds, move heavy Dart work off-thread |
| **Raster-thread jank** | Too slow to *render* the frame — expensive paint ops, shader compiles, huge images | Reduce clipping/blur/shadow cost, use Impeller, cache/resize images |

```dart
Widget build(BuildContext context) {
  final result = expensiveSyncComputation(); // ❌ 50ms+ blocking the UI thread
  return Text(result);
}
```

---

## What About Isolates?

> [!IMPORTANT]
> Isolates are **not** a stage in the rendering pipeline. They are a tool to move **CPU-heavy Dart work** off the UI isolate so it doesn't block frame production.

```dart
// ✅ Offload heavy computation without blocking the UI thread
final result = await compute(processLargeData, data);
```

- ✅ Good for: parsing large JSON, image processing, heavy calculations
- ❌ Does **not** move Flutter UI work (build / layout / paint) into the background
- Keep UI-related code on the UI isolate; offload only CPU-bound, non-UI Dart work

---

## 🎯 Key Interview Points

1. **UI-thread jank** = too slow to *produce* the frame.
2. **Raster-thread jank** = too slow to *render* the frame.
3. **Isolates** offload CPU-heavy Dart work — they are *not* a pipeline stage.
4. The UI thread never draws pixels; it only describes the scene via the Layer Tree.
5. **Impeller** removes most runtime shader-compilation jank that Skia was prone to.
