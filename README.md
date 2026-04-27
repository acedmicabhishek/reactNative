# React Native & Expo — Core Learning Path

A deep-dive research repo covering React Native from basics to custom engine architecture.
Each topic has its own dedicated guide.

---

## 📁 Folder Structure

```
reactNative/
├── 01-core-concepts/          ← React Native fundamentals
└── 02-architecture-and-performance/ ← Internals, engines, custom infra
```

---

## 📘 01 · Core React Native Concepts

| # | Topic | File |
|---|-------|------|
| 1 | Basic Components (`View`, `Text`, `Image`, `ScrollView`, `TextInput`) | [01-basic-components.md](./01-core-concepts/01-basic-components.md) |
| 2 | Styling & Layout (StyleSheet, Flexbox, Yoga, Dimensions) | [02-styling-and-layout.md](./01-core-concepts/02-styling-and-layout.md) |
| 3 | Interactivity, Touch & Gestures (RNGH, Reanimated Worklets) | [03-interactivity-and-gestures.md](./01-core-concepts/03-interactivity-and-gestures.md) |
| 4 | Lists & Virtualization (FlatList, SectionList, FlashList) | [04-lists-and-virtualization.md](./01-core-concepts/04-lists-and-virtualization.md) |
| 5 | State Management (Hooks, Zustand, Redux Toolkit, React Query) | [05-state-management.md](./01-core-concepts/05-state-management.md) |
| 6 | Platform-Specific Code (`Platform`, file extensions, permissions) | [06-platform-specific-code.md](./01-core-concepts/06-platform-specific-code.md) |

---

## ⚙️ 02 · Architecture & Performance (Under the Hood)

| # | Topic | File |
|---|-------|------|
| 1 | Threading Model (JS Thread, UI Thread, Shadow Thread) | [01-threading-model.md](./02-architecture-and-performance/01-threading-model.md) |
| 2 | The Legacy Bridge (JSON queue, bottlenecks, MessageQueue) | [02-legacy-bridge.md](./02-architecture-and-performance/02-legacy-bridge.md) |
| 3 | JSI — JavaScript Interface (HostObjects, direct C++ ↔ JS) | [03-jsi-javascript-interface.md](./02-architecture-and-performance/03-jsi-javascript-interface.md) |
| 4 | JS Engines — Hermes vs JSC (AOT bytecode, GC, V8 option) | [04-js-engines-hermes-jsc.md](./02-architecture-and-performance/04-js-engines-hermes-jsc.md) |
| 5 | Yoga Layout Engine (C++ Flexbox, Shadow Tree, dirty flags) | [05-yoga-layout-engine.md](./02-architecture-and-performance/05-yoga-layout-engine.md) |
| 6 | Rendering Pipeline (Render → Commit → Mount, Fabric phases) | [06-rendering-pipeline.md](./02-architecture-and-performance/06-rendering-pipeline.md) |
| 7 | New Architecture — Fabric & TurboModules (codegen, JSI modules, Concurrent React) | [07-new-architecture-fabric-turbomodules.md](./02-architecture-and-performance/07-new-architecture-fabric-turbomodules.md) |
| 8 | Memory Management (JS Heap, native memory, GC pressure, leaks) | [08-memory-management.md](./02-architecture-and-performance/08-memory-management.md) |
| 9 | JNI & C++ Bindings (Java↔C++, fbjni, ABI, custom native modules) | [09-jni-and-cpp-bindings.md](./02-architecture-and-performance/09-jni-and-cpp-bindings.md) |
| 10 | Metro Bundler (resolution, transformation, AST, Fast Refresh, source maps) | [10-metro-bundler.md](./02-architecture-and-performance/10-metro-bundler.md) |
| 11 | Android Build System (Gradle, R8, CMake, AAB, ProGuard) | [11-android-build-system.md](./02-architecture-and-performance/11-android-build-system.md) |
| 12 | Native Infrastructure (custom C++ modules, custom renderers, custom JS engines) | [12-native-infrastructure.md](./02-architecture-and-performance/12-native-infrastructure.md) |

---

## 🎯 Goal

Understand the full stack deeply enough to:
1. Identify exactly **where** performance is lost at each layer
2. **Bypass** framework limitations with custom C++ / JSI modules
3. Build **new rendering infrastructure** (custom Fabric renderer, custom JS engine)
4. Design a **better framework** than standard React Native where needed
