# The Rendering Pipeline in React Native

## Overview
The rendering pipeline is the complete journey from a JavaScript state change to pixels appearing on your device's screen. Understanding each phase reveals where performance is won or lost — and where a custom renderer could intervene.

---

## 1. The Three Phases (New Architecture / Fabric)

The Fabric renderer divides rendering into three distinct phases:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Phase 1: RENDER (JS Thread)                                                │
│  React reconciliation → new React Element Tree                              │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │  (C++ call via JSI)
┌─────────────────────────────────▼───────────────────────────────────────────┐
│  Phase 2: COMMIT (Background/UI Thread)                                     │
│  Cloning Shadow Tree → Yoga Layout → producing new Layout Tree              │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │  (scheduled on UI Thread)
┌─────────────────────────────────▼───────────────────────────────────────────┐
│  Phase 3: MOUNT (UI Thread)                                                 │
│  Diffing Layout Trees → creating/updating/deleting native Views             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Phase 1: Render (React Reconciliation)

**Thread:** JS Thread  
**Input:** State change, prop change, context change  
**Output:** New React Element Tree

```jsx
// A state change triggers re-render
const [count, setCount] = useState(0);
setCount(5); // triggers Phase 1
```

### What Happens
1. React calls the component function (or `render()`) to get the new tree of React Elements.
2. The **Fiber Reconciler** diffs the new tree against the previous tree.
3. It produces a **list of changes** (create, update, delete) — the "effect list" or "mutation list".
4. In Concurrent Mode: React can **pause this work** between components if a higher-priority update arrives.

### Key Data Structures
- **React Element:** Plain JS object `{ type, props, children }` — cheap to create.
- **Fiber Node:** Internal React object tracking each component's state, effects, and previous tree.
- **Work Loop:** React's main reconciliation loop that walks the Fiber tree.

### Performance Costs Here
- Component function re-execution
- Tree diffing (O(n) in number of components)
- `useMemo`, `React.memo`, `useCallback` checks

---

## 3. Phase 2: Commit (Shadow Tree + Layout)

**Thread:** Starts on JS Thread, most work in C++ (can run on background thread in New Architecture)  
**Input:** React mutation list from Phase 1  
**Output:** Committed Shadow Tree + Layout Tree (computed sizes & positions)

### What Happens
1. The Fabric C++ renderer receives the mutation list via JSI.
2. For each mutation:
   - **Create:** Clone a new `ShadowNode` from the component type's default node.
   - **Update:** Clone existing `ShadowNode` with new props.
   - **Delete:** Mark `ShadowNode` for removal.
3. A new **Shadow Tree** is built as an immutable tree of `ShadowNode` objects.
4. **Yoga runs** on this new Shadow Tree to compute layout.
5. The result is a new **Layout Tree** — each node annotated with `{x, y, width, height}`.

### Immutability
The Shadow Tree is **immutable** — every update produces a new tree via structural sharing (only changed nodes are copied; unchanged subtrees are reused). This enables:
- Concurrent renders without race conditions
- Safe background thread computation
- Easy tree diffing in Phase 3

### Performance Costs Here
- Yoga layout computation (C++, fast but O(n) in dirty nodes)
- Text measurement callbacks (native, can be slow for many unique strings)
- Shadow Node cloning (mostly fast due to structural sharing)

---

## 4. Phase 3: Mount (Native View Mutations)

**Thread:** UI Thread  
**Input:** Old Layout Tree + New Layout Tree  
**Output:** Created/updated/deleted native Views

### What Happens
1. The **Mount Manager** diffs the old and new Layout Trees.
2. For each change, it produces a **view mutation** — a command to the platform's View System:
   - `CREATE` → `new UIView()` / `new android.view.View()` 
   - `UPDATE` → `view.setBackgroundColor(...)`, `view.setFrame(...)`, etc.
   - `INSERT` → `parentView.addSubview(childView)` / `parentGroup.addView(childView)`
   - `REMOVE` → `parentView.removeFromSuperview()`
   - `DELETE` → release the view
3. View mutations are applied in a **single batch** — all changes for one frame happen at once.
4. After mutations are applied, the platform's compositor draws the new frame.

### The Commit Hook
In the New Architecture, you can intercept the Mount phase:
```cpp
// Register a MountingOverrideDelegate to intercept view mutations
class MyMountingDelegate : public MountingOverrideDelegate {
  bool shouldOverridePullTransaction() const override { return true; }

  Better pullTransaction(MountingTransaction& transaction, ...) override {
    // Inspect or modify mutations before they reach native views
    auto& mutations = transaction.getMutations();
    // ...
  }
};
```

---

## 5. The Full Pipeline: Concrete Example

**Scenario:** User presses a button → counter increments → list re-renders

```
[60ms ago] User touches screen
  → UI Thread: touch event detected → Pressable fires onPress
  → JS Thread: onPress callback → setCount(count + 1)

[PHASE 1: 0-8ms on JS Thread]
  → React calls Counter component
  → New Element: <Text>6</Text> (was 5)
  → Reconciler diffs: only the Text content changed
  → Effect list: [UPDATE TextNode, newProps = {children: '6'}]

[PHASE 2: 8-12ms in C++]
  → Fabric: clone Shadow Tree with new TextNode
  → Yoga: compute layout (only dirty TextNode remeasures)
  → New Layout Tree: TextNode {x:16, y:50, width:20, height:22}

[PHASE 3: 12-14ms on UI Thread]
  → MountManager diffs trees: TextNode content changed
  → Native mutation: UILabel.text = "6" (iOS) / TextView.setText("6") (Android)
  → Platform compositor: draw new frame

[~16ms] New frame displayed at 60fps ✓
```

If any phase takes too long and exceeds 16.67ms total, the frame is dropped (jank).

---

## 6. Legacy Architecture Pipeline (for comparison)

```
State change → React reconciliation (JS Thread)
  → UIManager.createView() calls serialized to JSON
  → Bridge queue → async → Native UIThread
  → Shadow Thread: Yoga layout
  → UI Thread: Apply views
  → Frame drawn
```

**Key differences from Fabric:**
- The transition from Phase 1 → Phase 2 goes over the async Bridge (adds latency).
- Shadow Tree is mutable (concurrent updates are risky).
- Layout tree is not diffed — full view updates are applied.
- Concurrent Mode features are unavailable.

---

## 7. Frame Budget Breakdown (60fps = 16.67ms)

Typical budget allocation for a well-optimized app:

| Phase | Budget | Notes |
|-------|--------|-------|
| JS Reconciliation | ~2-4ms | Small component trees |
| Yoga Layout | ~1-2ms | Only dirty nodes |
| Bridge/JSI transfer | ~0ms (JSI) / ~2ms (Bridge) | |
| View mutations (UI thread) | ~2-4ms | Depends on mutation count |
| GPU compositing | ~4-6ms | Platform handles this |
| **Total** | **~10-16ms** | Leaves margin for 60fps |

If you're consistently at the edge, profiling will show which phase is the bottleneck.

---

## 8. Custom Renderer: Intercepting the Pipeline

To build your own React Native renderer (like React Native Skia does), you need to:

1. **React Renderer (Phase 1):** Use `react-reconciler` package to create a custom reconciler host config. This intercepts all create/update/delete operations.

```js
const reconciler = ReactReconciler({
  createInstance(type, props) { /* create your own node type */ },
  appendChildToContainer(container, child) { /* add to your scene graph */ },
  commitUpdate(instance, updatePayload, type, oldProps, newProps) { /* update node */ },
  // ... 30+ more lifecycle methods
});
```

2. **Layout (Phase 2):** Either use Yoga directly via its JS or C++ bindings, or implement your own layout algorithm.

3. **Rendering (Phase 3):** Draw to a native surface using Skia, OpenGL, Metal, or Vulkan instead of UIKit/Android Views.

This is exactly how **react-native-skia** bypasses the native View system entirely to achieve GPU-accelerated 2D rendering within React Native.
