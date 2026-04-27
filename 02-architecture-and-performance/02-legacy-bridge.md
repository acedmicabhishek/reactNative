# The Legacy Bridge: Architecture, Bottlenecks & Limitations

## Overview
The "Bridge" is the original communication mechanism between the JavaScript world and the native (C++/Java/ObjC) world. It was the defining architectural choice of React Native from 2015 until the introduction of JSI and the New Architecture. Understanding it deeply is essential to understanding **why** the New Architecture was needed and **how** to design better performing systems.

---

## 1. What is the Bridge?

The Bridge is a **serialized, asynchronous message queue** that connects the JavaScript thread to the native (UI/Main) thread.

```
┌───────────────────────────────────────────────────────────┐
│                        JS Thread                          │
│  React components → createElement calls → VDOM diffs     │
└─────────────────────────────┬─────────────────────────────┘
                              │
                    JSON Serialization
                              │
                              ▼
┌───────────────────────────────────────────────────────────┐
│                     The Bridge                            │
│   Asynchronous, batched, JSON-serialized message queue    │
│                                                           │
│  Message: { module: 'UIManager', method: 'createView',    │
│             args: [1, 'RCTView', 100, {...styles}] }      │
└─────────────────────────────┬─────────────────────────────┘
                              │
                    JSON Deserialization
                              │
                              ▼
┌───────────────────────────────────────────────────────────┐
│                     UI Thread (Native)                    │
│   Native modules → ViewManager → actual UIView / View    │
└───────────────────────────────────────────────────────────┘
```

---

## 2. How the Bridge Works: Step by Step

### Startup
1. Native app launches, starts the JS runtime (Hermes/JSC).
2. Native registers all available modules in a **Module Registry** (a JSON map of module names → method signatures).
3. JS loads the module registry and generates proxy objects for each native module.

### JS → Native Call (e.g., `NativeModules.CameraModule.takePicture()`)
1. JS calls the method on the proxy object.
2. The call is serialized: `{ moduleId: 12, methodId: 3, callId: 7, args: [{quality: 0.9}] }`.
3. The serialized message is placed in the **outgoing queue**.
4. The **MessageQueue** batches outgoing calls (flushes every ~5ms or when the queue gets large).
5. The native side dequeues the message, deserializes it, looks up `moduleId:12, methodId:3`, and calls the Java/ObjC method.
6. The method runs on the UI thread (or a module background thread).
7. If there's a return value or callback: it's serialized back and sent across the bridge in the **incoming queue**.

---

## 3. Why Serialization is Expensive

JSON serialization/deserialization is O(n) in the size of the data.

**Example: Updating a list of 100 items:**
```json
// Each UIManager.updateView call must serialize the entire style object
{ "module": "UIManager", "method": "updateView", "args": [42, "RCTView", {"backgroundColor": "#fff", "borderRadius": 8, "padding": 16, ...}] }
```

For a 60fps animation updating every frame, the bridge must serialize+deserialize 60 messages per second, each potentially containing a full style object. **This is the primary source of animation jank in legacy RN.**

### Benchmarks (approximate)
| Operation | Legacy Bridge | JSI (New Architecture) |
|-----------|--------------|------------------------|
| Simple JS→Native call | ~0.2ms+ | ~0.01ms |
| Passing 1MB of data | serialization + deserialization overhead | direct memory reference, near-zero |
| Synchronous return | **impossible** | natively supported |

---

## 4. The MessageQueue

```
JS side:
  MessageQueue.enqueueNativeCall(moduleID, methodID, args, onSuccess, onFail)

Native side:
  MessageQueue.callFunctionReturnFlushedQueue(moduleName, method, args)
  // returns all pending JS→native messages
```

**Batching:** The MessageQueue does NOT send messages one by one. It batches them and flushes:
- After every frame (at 60fps cadence)
- When explicitly flushed (`flushQueue()`)
- When the message is urgent (e.g., `callImmediates`)

**Implication:** There is inherent latency between a JS state change and the native UI update. This latency is at minimum one frame (16.67ms) but can be several frames if the JS thread or UI thread is busy.

---

## 5. The Shadow Queue and UIManager

The UIManager is the most heavily used native module. Every `View` creation, update, and deletion goes through it.

```
JS: <View style={{ flex: 1, backgroundColor: 'red' }} />
  ↓
React Reconciler: createNode(type='RCTView', props={...})
  ↓
UIManager.createView(reactTag, viewName, rootTag, props)  [BRIDGE CALL]
  ↓
Native UIManager receives call, creates a Shadow Node (C++ / Yoga)
  ↓
UIManager.manageChildren() / updateView() [additional BRIDGE CALLS]
  ↓
Yoga computes layout
  ↓
Native ViewManager inflates actual native View
```

Every prop update (`setNativeProps` or state change) triggers additional UIManager bridge calls.

---

## 6. `setNativeProps` — A Bridge Optimization Hack

`setNativeProps` allows bypassing React's reconciliation to update a native view directly from JS, reducing the amount of diffing and bridge traffic.

```jsx
const viewRef = useRef(null);

// Instead of setState (full React re-render + bridge serialization)
viewRef.current.setNativeProps({
  style: { opacity: 0.5, backgroundColor: '#ff0000' },
});
```

**⚠️ When to use it:** Only in animation hot paths where Reanimated isn't an option. It bypasses React's source of truth, making components hard to reason about.

---

## 7. The `__fbBatchedBridge` Object

In old React Native builds, you could inspect the bridge directly via the global `__fbBatchedBridge` object in the Chrome debugger:

```js
// In Chrome DevTools (remote JS debugging)
window.__fbBatchedBridge._queue
// Shows pending messages waiting to be flushed across the bridge
```

This is now deprecated with JSI but historically was useful for debugging bridge overload.

---

## 8. Key Limitations of the Bridge

1. **No synchronous calls.** JS cannot block and wait for a native result. Everything must be callback/promise-based.
2. **Serialization overhead.** Complex objects (large arrays, nested styles) are expensive to serialize.
3. **No shared memory.** JS and native cannot share data structures — copies are always made.
4. **Animation bottleneck.** Per-frame animation updates require 60 bridge crossings per second.
5. **Single batched queue.** All native module calls share one queue — a flood of UIManager calls can delay NativeModules calls and vice versa.

---

## 9. The Bridge in 2024+

The Bridge is being deprecated in favor of JSI. However:
- Many third-party libraries still use the Bridge (via the `NativeModules` API).
- Even in New Architecture mode, the Bridge is available as a compatibility layer.
- Understanding it is still required for reading legacy code, debugging library issues, and understanding how RN evolved.

---

## 10. If You Were to Build a Better Bridge

Areas to improve (and what the New Architecture addresses):

| Problem | Solution in New Architecture | Alternative Approach |
|---------|------------------------------|----------------------|
| Serialization overhead | JSI — direct C++ references | Shared memory (FlatBuffers, Cap'n Proto) |
| No sync calls | JSI supports sync calls | Was never possible in legacy |
| Queue flooding | TurboModules — lazy loading | Priority-based queue |
| No concurrent rendering | Fabric + Concurrent React | N/A in legacy |
| Single-threaded JS | Not solved yet | WASM threading, JS Workers |
