# The Threading Model in React Native

## Overview
React Native is inherently **multi-threaded**. Understanding which code runs on which thread is the single most important concept for diagnosing performance issues. A frame drop almost always means work that should be on a background thread is happening on the JS or UI thread.

---

## 1. The Threads

### 1a. Main Thread (UI Thread / Native Thread)
- The **single, precious** thread that the OS uses to draw pixels to screen.
- Runs at **60fps (16.67ms per frame)** or **120fps (8.33ms per frame)** on ProMotion displays.
- Any work here that takes longer than one frame budget blocks rendering → **jank**.
- Handles: touch events, measuring/laying out native views, calling native module methods, drawing to the screen.

### 1b. JavaScript Thread
- The thread where your React/JS code runs (inside the Hermes or JSC engine).
- Single-threaded (JS is single-threaded by design).
- If this thread is blocked (heavy computation, synchronous API calls), the UI won't respond to touch events or schedule new renders.
- Handles: React reconciliation, business logic, state updates, API calls.

### 1c. Shadow Thread (Background Thread)
- A C++ thread dedicated to **Yoga layout computation**.
- Receives the React component tree from the JS thread.
- Computes exact sizes and positions for every node using the Yoga engine.
- Sends computed layout to the UI thread to create/update native views.
- Invisible to JS developers — you don't interact with it directly.

### 1d. Native Module Threads
- Each native module (Camera, GPS, Bluetooth, custom modules) may spawn its own background thread.
- Network requests (`fetch`, `axios`) are dispatched to a thread pool managed by the OS.
- AsyncStorage, File I/O, Crypto — all run off-thread to avoid blocking the JS thread.

---

## 2. Thread Communication (Legacy Architecture)

In the legacy Bridge architecture, threads communicate **asynchronously** via a serialized message queue.

```
JS Thread
   │
   │ Serialize to JSON ─────────────────────────────────┐
   │ (e.g., { module: 'UIManager', method: 'createView' }) │
   │                                                     │
   ▼                                                     ▼
 Bridge (Message Queue)                         Bridge (Message Queue)
   │                                                     │
   │ Deserialize JSON                    Deserialize JSON │
   ▼                                                     ▼
UI Thread (execute native call)          JS Thread (receive native event)
```

**This is the core bottleneck of the legacy architecture:**
- Every call between JS and native (including UI updates!) must be serialized to JSON, queued, dispatched, and deserialized.
- This is asynchronous — JS cannot wait for a native result synchronously.
- If many messages pile up, the queue introduces visible latency.

---

## 3. Thread Communication (New Architecture / JSI)

With JSI (JavaScript Interface), the JS thread can hold **direct C++ references** to native objects:

```
JS Thread
   │
   │ Direct C++ function call (no serialization!)
   │ (hostObject.doSomething(args))
   │
   ▼
Native (UI Thread or its thread)
   │
   │ Direct return value (synchronous if on same thread)
   ▼
JS Thread
```

No JSON serialization. No queue. Synchronous or asynchronous depending on need.

---

## 4. Thread Interaction Diagram (Legacy)

```
┌──────────────┐       JSON queue      ┌──────────────┐
│  JS Thread   │ ───────────────────▶  │  UI Thread   │
│              │                       │              │
│  React       │ ◀─────────────────── │  Native      │
│  reconcile   │     Events / layout   │  Views       │
│  State mgmt  │       results         │  Touch events│
│  API calls   │                       │  Drawing     │
└──────────────┘                       └──────────────┘
        │                                      │
        │                              ┌───────┴──────────┐
        │                              │  Shadow Thread   │
        │ sends VDOM diff              │  (Yoga C++)      │
        └────────────────────────────▶ │  Layout compute  │
                                       └──────────────────┘
```

---

## 5. Diagnosing Thread Issues

### Tool: Systrace / Android Profiler
On Android, use `adb` + Android Studio's profiler or Chrome's Systrace to see what's running on each thread frame-by-frame.

```bash
# Capture Systrace on Android
adb shell atrace --async_start -c -b 32768 gfx view sched
# ... reproduce the jank ...
adb shell atrace --async_stop
adb pull /data/local/tmp/trace.html .
# Open in Chrome -> about:tracing
```

### Tool: React DevTools Profiler
Shows which React components rendered and how long they took — but only gives JS-thread timing.

### Tool: Flipper (Legacy) / DevTools Network Inspector
Inspect bridge traffic in the legacy architecture — see which JS ↔ native messages are flooding the queue.

### Symptom → Root Cause Map

| Symptom | Likely Culprit |
|---------|----------------|
| Gestures feel laggy | JS thread is busy (animation computed in JS, not Worklet) |
| App freezes on button press | Expensive synchronous code on JS thread |
| Blank frames during scroll | Virtualization rendering on JS thread too slow; increase `maxToRenderPerBatch` |
| Slow initial load | Too much JS parsing / big bundle |
| Native module responses are slow | Native module spawning work on main thread; should use background thread |

---

## 6. Best Practices

1. **Never block the JS thread.** Move heavy computation to a Worker (`react-native-multithreading` or `expo-task-manager`).
2. **Use Reanimated Worklets** for animations — they run directly on the UI thread.
3. **Use RNGH (React Native Gesture Handler)** — gestures are processed on the UI thread, not routed through JS.
4. **Heavy computation** (sorting 10k items, parsing large JSON): use `InteractionManager.runAfterInteractions()` or a native module.
5. **Image decoding** is done natively (off both JS and UI threads). The bottleneck is usually re-layout caused by `undefined` dimensions.

---

## 7. The Future: Concurrent React + New Architecture

The New Architecture enables React's **Concurrent Mode** in React Native:
- React can pause, abort, and resume rendering work.
- High-priority updates (user input) can interrupt low-priority renders (list updates).
- `useTransition` and `useDeferredValue` become meaningful on mobile.
- Long React renders can be broken up across multiple frames without blocking the UI thread.

This is the biggest threading improvement in React Native's history.
