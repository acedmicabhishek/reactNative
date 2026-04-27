# Yoga Layout Engine

## Overview
Yoga is the **C++ cross-platform Flexbox implementation** that powers React Native's layout system. It's not just a library — it's the engine that translates your JavaScript style objects into exact pixel coordinates that native views use to position themselves on screen. Understanding Yoga is essential for diagnosing layout performance issues and for building custom rendering systems.

---

## 1. What is Yoga?

Yoga is an open-source C++ library developed by Meta that implements the **CSS Flexbox specification**. It runs on:
- React Native (mobile)
- React Native for Windows / macOS
- Litho (Meta's Android UI framework)
- ComponentKit (Meta's iOS UI framework)
- Yoga is also available as a standalone library

**Key design goals:**
- Platform-independent (same layout results on Android and iOS)
- High performance (pure C++, no allocations in hot paths)
- Small and embeddable

---

## 2. Where Yoga Runs in React Native

In the legacy architecture, Yoga runs on the **Shadow Thread** (a dedicated background thread):

```
JS Thread
  │ Creates shadow node tree (mirrors React component tree)
  │ Sends to Shadow Thread
  ▼
Shadow Thread (C++)
  │ Yoga layout pass runs here
  │ Computes size and position of every node
  │ Sends results to UI Thread
  ▼
UI Thread
  │ Applies computed layout to native Views
  ▼
Screen
```

In the New Architecture (Fabric), the Shadow Thread is replaced by the **Fabric C++ renderer**, but Yoga still does the layout work.

---

## 3. Yoga's Data Model: The Shadow Tree

Every React Native component has a corresponding **YogaNode** in C++. These form the **Shadow Tree** — a parallel tree that mirrors the React component tree but lives in C++.

```cpp
// YogaNode represents a single layout node
YGNodeRef node = YGNodeNew();

// Set style properties
YGNodeStyleSetFlexDirection(node, YGFlexDirectionColumn);
YGNodeStyleSetWidth(node, 375);      // in dp
YGNodeStyleSetHeight(node, YGUndefined); // auto-sized
YGNodeStyleSetPadding(node, YGEdgeAll, 16);
YGNodeStyleSetAlignItems(node, YGAlignCenter);

// Build the tree
YGNodeRef child = YGNodeNew();
YGNodeStyleSetFlex(child, 1);
YGNodeInsertChild(node, child, 0);

// Run layout
YGNodeCalculateLayout(node, 375, 812, YGDirectionLTR);

// Read results
float left   = YGNodeLayoutGetLeft(child);    // x position
float top    = YGNodeLayoutGetTop(child);     // y position
float width  = YGNodeLayoutGetWidth(child);   // computed width
float height = YGNodeLayoutGetHeight(child);  // computed height
```

---

## 4. The Layout Algorithm (Three Phases)

Yoga implements a **3-phase layout algorithm**:

### Phase 1: Measure
- Visits each node and determines its content size.
- For leaf nodes with `Text`: invokes a native text measurement function to get actual text dimensions.
- For leaf nodes with `Image`: uses the image's intrinsic size or explicit style.
- For container nodes: deferred to Phase 3.

### Phase 2: Flex (Main Pass)
- Distributes space along the main axis according to `flex`, `flexGrow`, `flexShrink`.
- Handles `flexWrap` — creates additional flex lines if needed.
- Applies `justifyContent` distribution.

### Phase 3: Align (Cross Axis)
- Aligns items on the cross axis according to `alignItems` and `alignSelf`.
- For `alignItems: 'stretch'`: computes the cross-axis size for children that didn't specify a cross-axis size.

---

## 5. Yoga Style Properties → C++ Mapping

| JS StyleSheet | Yoga API | Notes |
|--------------|----------|-------|
| `flexDirection: 'row'` | `YGFlexDirectionRow` | Default is `column` in RN |
| `justifyContent: 'center'` | `YGJustifyCenter` | |
| `alignItems: 'stretch'` | `YGAlignStretch` | |
| `flex: 1` | `YGNodeStyleSetFlex(node, 1)` | |
| `width: 100` | `YGNodeStyleSetWidth(node, 100)` | In dp |
| `width: '50%'` | `YGNodeStyleSetWidthPercent(node, 50)` | |
| `position: 'absolute'` | `YGPositionTypeAbsolute` | |
| `gap: 12` | `YGNodeStyleSetGap(node, YGGutterAll, 12)` | New in Yoga 2 |

---

## 6. The Measure Function (Custom Leaf Nodes)

For leaf nodes that need custom measurement (Text, Image, video frames), Yoga allows registering a **measure function** — a C++ callback that returns the node's intrinsic size:

```cpp
// Register a measure function for a Text node
YGNodeSetMeasureFunc(textNode, [](
  YGNodeConstRef node,
  float availableWidth,
  YGMeasureMode widthMode,
  float availableHeight,
  YGMeasureMode heightMode
) -> YGSize {
  // Call native text measurement
  // (uses platform font metrics to compute actual text bounds)
  const char* text = (const char*)YGNodeGetContext(node);
  NSString* nsText = [NSString stringWithUTF8String:text];
  CGSize size = [nsText sizeWithAttributes:@{NSFontAttributeName: [UIFont systemFontOfSize:16]}];
  return { (float)ceil(size.width), (float)ceil(size.height) };
});
```

This is why `Text` components in React Native layout correctly — their actual rendered size is computed via native text measurement APIs.

---

## 7. Yoga and the Dirty Flag System

Yoga uses a **dirty flag** to avoid recomputing layout for subtrees that haven't changed.

```
State change → React reconciler marks affected nodes dirty
                    ↓
Yoga marks those YogaNodes dirty (and their ancestors)
                    ↓
Layout pass: only recomputes nodes that are dirty
                    ↓
Unchanged subtrees: skip (use cached layout results)
```

This makes layout incremental — a state change to one component doesn't force a full-tree layout recompute.

**Custom Renderer Implication:** If you're building a custom renderer on top of React Native's React reconciler (like the Fabric model), you need to propagate dirty flags correctly to Yoga nodes to maintain this optimization.

---

## 8. Layout Performance Profiling

### Detecting Layout Bottlenecks

```bash
# Android: Use Systrace to see time spent in layout pass
python $ANDROID_HOME/platform-tools/systrace/systrace.py \
  --time=10 -o trace.html view gfx

# In Chrome -> about:tracing -> Load trace.html
# Look for: YogaLayout, RCTBatchedBridge, Shadow Thread activity
```

### `onLayout` Callback — Measuring Rendered Components

```jsx
<View onLayout={(event) => {
  const { x, y, width, height } = event.nativeEvent.layout;
  // Called after Yoga computed layout AND native view was measured
  // Avoid triggering state updates here (causes re-render + re-layout)
}}>
```

**⚠️ Warning:** Calling `setState` inside `onLayout` triggers another full layout pass. Use it only for read-only measurement.

---

## 9. Yoga's Limitations and Where It Breaks Down

1. **No CSS Grid support.** Yoga only implements Flexbox. Grid layouts require manual calculation or nested flex containers.
2. **Percentage widths require a known parent.** If a parent has no explicit size, `width: '50%'` for a child can produce unexpected results.
3. **Text measurement is expensive.** Every unique string with unique styles requires a native text measurement call. Long lists of `Text` components with many style variations are layout-heavy.
4. **Single-threaded within Shadow Thread.** Yoga runs all layout computation serially on one thread. Deeply nested trees with many dirty nodes will take longer.

---

## 10. Yoga 2.0 (Current)

Released in 2023, Yoga 2.0 brings:
- **`gap` property support** (previously missing from CSS Flexbox spec in Yoga)
- **Errata mode** — opt-in to match W3C spec exactly vs. legacy RN behavior
- **Improved correctness** for edge cases
- **Better C++ API** with namespaced types

```cpp
// Yoga 2 uses namespaced types
facebook::yoga::Node node;
node.setStyle(facebook::yoga::Style{
  .flexDirection = facebook::yoga::FlexDirection::Column,
  .alignItems = facebook::yoga::Align::Center,
});
```

---

## 11. Replacing Yoga: Could You?

If you were to build a custom layout engine for React Native:

- You'd need to implement the `YogaNode`-equivalent interface that Fabric calls into during `shadowNodeClone()` and `measure()`.
- The Fabric C++ layer uses `ShadowNode` → `LayoutableShadowNode` → calls `layout()` on each node.
- Implementing a GPU-accelerated layout engine (running on a render thread with SIMD-optimized Flexbox) could significantly outperform Yoga's single-threaded CPU layout.

This is a legitimate area of research for building high-performance custom RN-compatible frameworks.
