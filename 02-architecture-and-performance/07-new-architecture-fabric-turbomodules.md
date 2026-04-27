# The New Architecture: Fabric & TurboModules

## Overview
The "New Architecture" is the umbrella term for a complete redesign of React Native's internals, enabled since RN 0.68 and made the default in RN 0.74+. It replaces the asynchronous, serialization-heavy Bridge with a synchronous JSI-based system, enabling concurrent rendering, better native performance, and more predictable behavior.

---

## 1. What's Included in the New Architecture

| Component | Replaces | What it Does |
|-----------|---------|--------------|
| **Fabric** | UIManager + Shadow Thread | C++ renderer: manages Shadow Tree, Yoga layout, View mutations |
| **TurboModules** | NativeModules (Bridge) | JSI-based native modules with lazy loading |
| **Codegen** | Manual bindings | Auto-generates C++ / ObjC / Java glue code from TypeScript specs |
| **JSI** | Bridge serialization | Direct C++ ↔ JS communication layer |

---

## 2. Fabric: The New Renderer

Fabric is a complete C++ rewrite of the rendering pipeline. Its key improvements over the legacy UIManager:

### A) Synchronous Layout Reads
In the legacy architecture, reading layout values (size, position) from JS was impossible synchronously — you had to wait for `onLayout` callbacks.

With Fabric, the Shadow Tree is accessible from JS via JSI, enabling synchronous reads:
```jsx
// New Architecture: get layout synchronously
const ref = useRef(null);
ref.current.measure((x, y, width, height) => {
  // In New Arch, this is synchronous on the same frame
  // In Legacy Arch, this was a cross-bridge async call
});
```

### B) Concurrent React Support
Fabric enables React's Concurrent Mode features:
- **`useTransition`:** Mark state updates as non-urgent — React can defer and interrupt them.
- **`useDeferredValue`:** Defer re-rendering an expensive subtree until the UI thread is idle.
- **Priority-based rendering:** User input updates preempt background data updates.

```jsx
import { useTransition } from 'react';

const [isPending, startTransition] = useTransition();

// This large list update is interruptible — won't block user input
startTransition(() => {
  setFilteredItems(hugeListFilter(allItems, query));
});
```

### C) Immutable Shadow Tree
Every state update creates a new Shadow Tree (structural sharing). Benefits:
- Safe concurrent reads from multiple threads.
- Easy rollback (just switch back to previous tree).
- No synchronization locks needed for read access.

### D) The Fabric C++ Layer

```
React Element Tree (JS, virtual DOM)
         │ JSI call
         ▼
ShadowNode Tree (C++, immutable)
         │ Yoga layout pass
         ▼
Layout Tree (C++, computed positions)
         │ Diffing
         ▼
MutationList (C++)
         │ UI Thread dispatch
         ▼
Native View Tree (UIView / android.view.View)
```

Key C++ types:
- `ShadowNode` — abstract node in the shadow tree
- `ConcreteComponentDescriptor<T>` — factory for creating ShadowNodes of type T
- `MountingCoordinator` — coordinates when to apply mutations to UI thread
- `UIManager` (C++) — the Fabric-side UIManager (very different from the legacy UIManager)

---

## 3. TurboModules: The New Native Module System

### Problems with Legacy NativeModules

1. All native modules were **initialized at startup**, even unused ones (slow startup).
2. All calls went over the async Bridge (serialization overhead).
3. No type safety between JS and native.

### TurboModules Solution

**Lazy Loading:** A TurboModule's C++ object is only created when JS first accesses it.

```
// Legacy: ALL modules initialized at app start
ModuleA.init(), ModuleB.init(), ModuleC.init(), ... // even if never used!

// TurboModules: module initialized on first use
const camera = TurboModuleRegistry.get('NativeCameraModule');
// ↑ Only NOW does CameraModule C++ code execute
```

**JSI-based calls:** Direct C++ invocation via JSI (no Bridge, no serialization).

**Type-safe specs:** A TypeScript spec is the single source of truth.

### Codegen Workflow

```
1. Write TypeScript spec:
   NativeCameraModule.ts
     → Spec interface with method signatures

2. Run Codegen:
   yarn react-native codegen
     → Generates C++ abstract class: NativeCameraModuleCxxSpec.h
     → Generates Android: NativeCameraModuleJavaSpec.java (JNI glue)
     → Generates iOS: RCTNativeCameraModuleSpec.h (ObjC protocol)

3. Implement the spec:
   NativeCameraModule.cpp
     → inherits NativeCameraModuleCxxSpec
     → implements the actual camera capture logic

4. Register:
   ReactPackage.createTurboModules() → returns NativeCameraModule instance
```

### TypeScript Spec Example

```typescript
// NativeCameraModule.ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  // Synchronous method — only possible with TurboModules!
  isAvailable(): boolean;
  // Async method
  takePicture(options: {quality: number; flash: boolean}): Promise<string>;
  // Event emitter (replaces addListener pattern)
  addListener(eventName: string): void;
  removeListeners(count: number): void;
}

export default TurboModuleRegistry.getEnforcing<Spec>('NativeCameraModule');
```

### C++ Implementation Skeleton

```cpp
// NativeCameraModule.h
#pragma once
#include <NativeCameraModuleCxxSpec.h>  // generated by codegen

namespace facebook::react {

class NativeCameraModule : public NativeCameraModuleCxxSpec<NativeCameraModule> {
public:
  NativeCameraModule(std::shared_ptr<CallInvoker> jsInvoker);

  bool isAvailable(jsi::Runtime& rt);

  jsi::Value takePicture(
    jsi::Runtime& rt,
    jsi::Object options  // typed by codegen
  );
};

} // namespace
```

---

## 4. Codegen in Depth

Codegen is the **build-time tool** that reads TypeScript specs and generates native binding code.

```bash
# Run codegen manually (normally done automatically by the build system)
cd android && ./gradlew generateCodegenArtifactsFromSchema
cd ios && pod install  # also triggers codegen for iOS

# Output locations:
android/app/build/generated/source/codegen/
  └── java/com/facebook/react/viewmanagers/  # ViewManager bindings
  └── jni/react/renderer/components/         # C++ ShadowNode specs

ios/build/generated/ios/
  └── FBReactNativeSpec/                     # TurboModule ObjC protocols
  └── RCTThirdPartyFabricComponentsProvider/ # Custom component registration
```

**What codegen generates:**
1. **TurboModule specs:** C++ abstract class with pure virtual methods (you implement them).
2. **ViewManager specs:** C++ `ComponentDescriptor` and `Props` structs for Fabric components.
3. **JNI glue:** Java/Kotlin adapter classes that proxy between Java native methods and C++.
4. **ObjC protocols:** iOS `RCTBridgeModule` / `RCTTurboModule` conformance declarations.

---

## 5. Migration: Legacy Bridge → New Architecture

For existing library authors, the bridge is still supported as a **compatibility layer** even in New Architecture mode. But to get full benefits:

### Backwards-compatible approach (Interop Layer)
```jsx
// Libraries not yet migrated work via the Legacy Module Interop Layer
// They still use the Bridge internally but can run alongside TurboModules
// Performance: no improvement, but the app doesn't break
```

### Full migration
1. Add TypeScript spec files.
2. Run codegen.
3. Rewrite native module to implement generated spec.
4. Set `newArchEnabled=true` in `gradle.properties`.

---

## 6. Enabling New Architecture

**Android:**
```properties
# android/gradle.properties
newArchEnabled=true
```

**iOS:**
```ruby
# ios/Podfile
ENV['RCT_NEW_ARCH_ENABLED'] = '1'
```

**Expo:**
```json
// app.json
{
  "expo": {
    "newArchEnabled": true
  }
}
```

---

## 7. Real Performance Gains (Measured)

| Scenario | Legacy | New Architecture |
|----------|--------|-----------------|
| Initial module load | All modules at startup | Lazy (often 200-300ms saved) |
| Animation at 60fps | JS thread must be free | UI thread worklets (JS thread can be busy) |
| Layout measurement | Async (1+ frame delay) | Synchronous |
| Large list scroll | Frequent blank frames | Fewer (Concurrent Mode can prioritize scroll) |
| Native module call overhead | ~0.2-1ms (bridge) | ~0.01ms (JSI) |

---

## 8. The Opportunity: What's Still Left to Build

Even with the New Architecture, there are opportunities for further optimization:

1. **Layout on GPU:** Yoga runs on CPU. A SIMD-optimized or GPU-accelerated layout engine could be 10-100x faster for large trees.

2. **Compiled component trees:** Instead of re-running React reconciliation on every state change, precompile component trees into native view templates that can be instantiated without JS execution.

3. **Ahead-of-time View creation:** Predict which views will appear and pre-create them in native before they're needed (prefetch + inflate).

4. **Custom Fabric Renderer:** Build a non-View renderer (OpenGL/Skia/Vulkan) that Fabric can talk to — bypassing UIKit/Android Views entirely for custom drawing components.

5. **Off-main-thread composition:** Android 12+ and iOS support off-main-thread rendering. A custom Fabric renderer could push composition work off the UI thread.
